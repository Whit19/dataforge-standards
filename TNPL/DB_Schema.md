# TNPL — DB Schema
> **Protocol:** Load MASTER_CLAUDE_PROTOCOL.md before this file.
> Moved out of TechnicalArchitecture.md (2026-10-14) to keep that file lean — it's fetched every session; this schema reference is linked from it instead.

## Data model (Firestore)

```
players/{playerId}
  name, email, phone, role: 'player' | 'staff', active, isAdmin
  currentElo, seasonStartElo, eloIsDefault  // player role only; missing = pairing defaults to 1500
  authUid                               // set on first sign-in once matched
  inviteStatus: not_invited | invited | accepted | declined
  invitedAt, declineTokenHash           // set while inviteStatus is 'invited' (fresh on every re-invite, TP-071)
  declinedAt                            // set only by the POST decline (TP-070), cleared on any re-invite (TP-071)
  firstSignInAt                         // set once, on the first successful sign-in — the durable
                                         // "has this player signed in" marker (TP-038), independent
                                         // of inviteStatus (which a later re-invite can otherwise touch)
  reminderSentAt                        // sendInvites mode: 'reminder' only (TP-073) — distinct from
                                         // weeks.{availabilityEmailSentAt,reminderSentAt}, which guard
                                         // the weekly-automation emails, not this one. Also drives a 24h
                                         // reminded_recently duplicate-send skip, server and client,
                                         // bypassable only server-side via a force param (TP-076)
  contactPreference: 'email' | 'push' | 'both'
  hidePhone, hideEmail: boolean         // self-writable; drives directory.phone/email, not a players-level
                                         // hide since Firestore rules can't hide individual fields (TP-056)

directory/{playerId}                    // public-safe view of players; one doc per roster person
  name, sortName, role, active, isAdmin, listed: boolean
  phone, email                          // present only when listed AND not hidden; otherwise null
  hidePhone, hideEmail                  // mirrored from players, for the Players page's own UI
  updatedAt
  // Server-write only (Cloud Firestore triggers in functions/players/directorySync.js, on BOTH
  // players writes and sessionEnrollment writes). `listed` = active AND signed in (firstSignInAt)
  // AND opted into the active season, for player role; staff are listed once active. Everyone else
  // still gets a name-only entry (listed:false, no phone/email) so Rankings/History/match screens
  // can show a name (TP-056).

playerRatings/{playerId}                // Elo fields only; players only, never staff (TP-016)
  currentElo, seasonStartElo, eloIsDefault
  updatedAt
  // Server-write only, same triggers as directory above (TP-056). currentElo/
  // seasonStartElo write null, never undefined, when the source players field
  // is missing (TP-058) — a guest created via the change-request "replace"
  // flow has no seasonStartElo, and Firestore rejects undefined outright.

playerLinks/{authUid}
  playerId
  // Reverse index, auth UID -> playerId. Firestore security rules can only
  // do get() by a known document path, not arbitrary queries — this is what
  // lets isAdmin()/isPlayerRole() rules helpers resolve "which player is the
  // caller" from request.auth.uid alone. Written by the auth Cloud Functions
  // (verifySignInCode / linkAccount) alongside inviteStatus/authUid/
  // firstSignInAt on the matched player doc. (TP-014)

seasons/{seasonId}                      // e.g. '2026-27'
  name, startDate, endDate, status: 'active' | 'complete'   // 'complete' added for history import (TP-052)
  imported: boolean                     // true for a season written by importSeasonHistory.js, never players
  kFactor, pairingFlagThreshold (TP-018), slotFairnessTolerance (TP-021),
  newPlayerElo, guestElo, provisionalUnderSets, carryOverFactor, carryOverSetsPivot,  // carryOver* stored,
                                         // no consuming logic yet — season close isn't built (TP-054)
  dinnerLeadMinutes, automation: { availabilityOpens, reminder, draftBuilt, pairingsSent, finalized }
  defaultTimeSlots
  // seasons is now server-write only (allow write: if false) — the setupSeason / Settings callables
  // (Admin SDK) are the only writers; no client write path depended on direct access (TP-054).

sessions/{seasonId}-s{number}           // a season has two sessions (TP-029)
  seasonId, number, name: "Session 1", startDate, endDate
  status: not_started | open | closed

sessionEnrollment/{sessionId}_{playerId}
  playerId, sessionId, seasonId
  choice: full | session_1 | session_2 | none    // the player's one answer, copied onto every session doc
  optedIn: boolean                               // derived per session from choice + that session's number
  choiceSetBy: 'player' | 'admin'                // 'player' clears choiceSetByUid; 'admin' sets it (TP-074)
  choiceSetByUid                                 // the admin's own auth uid — admin path only
  updatedAt
  // Written only by the setSeasonSignup callable, which keeps every session
  // doc for a player consistent in one batch. setSeasonSignup takes an
  // optional playerId so an admin can record a choice for a player who has
  // never signed in — classify() honors it either way (TP-074).

weeks/{weekId}                          // weekId is the ISO date, e.g. '2026-10-15'
  seasonId, sessionId, date, weekNumber, sessionWeekNumber
  status: draft | availability_open | pairing_draft | matches_set | in_progress | complete | skipped | cancelled
  // 'cancelled' is RESERVED for a future whole-week cancel before pairings are
  // sent — nothing sets it yet. Per-slot cancel (below) never changes
  // week.status at all; it's a completely separate mechanism (TP-061).
  timeSlots: [{ id, label: "6:00 PM", start, courtsAvailable: 2 }, ...]   // slot order = array order
  customTimes: boolean                  // true once this week's slots were edited away from the season default (TP-053)
  pairingHold: boolean                  // admin can hold the Wednesday auto-send
  pendingNotify: []                     // players to notify after a post-send admin edit
  dinnerSpecial: string | null          // optional, <=120 chars, shown as "This week's special:" everywhere (TP-055)
  extraQuestion: { text } | null        // optional per-week yes/no question, <=80 chars (TP-055)
  automation: { availabilityOpensAt, reminderAt, draftBuiltAt, pairingsSentAt, finalizedAt }  // Timestamps
  availabilityEmailSentAt, reminderSentAt          // set by weeklyAutomation, guards a double-send
  lockReminderSentAt                               // Tuesday-reminder guard (TP-047)
  lockedAt, lockedBy, lockMode: 'manual' | 'auto' | 'import'  // 'import' set by importSeasonHistory.js (TP-052)
  unlockedAt, unlockedBy                           // set by unlockWeekInternal
  cancelledSlots: { [slotId]: { at, by, reason } } // per-slot cancel after pairings are sent (TP-061);
                                                    // reason is a trimmed string <=60 chars, or null
  watchdogAlerts: { [step]: Timestamp }            // automationWatchdog's "already alerted for this
                                                    // step" markers — availability/reminder/draft/send/
                                                    // finalize — so an overdue alert never repeats (TP-066)
  // weeks is now server-write only (allow write: if false), same audit as seasons above (TP-054).

availability/{weekId}_{playerId}
  weekId, playerId                      // TPI-005: written on every doc; the pairing callable can
                                         // still find a week's docs by document-ID prefix "{weekId}_"
  canPlay, canPlayTwo, blockedSlotIds[], preferredSlotIds[], notes
  extraAnswer: boolean | null           // answer to that week's weeks.extraQuestion, if any (TP-055);
                                         // only asked of/counted for players who said they're playing
  respondedAt                           // set on the FIRST answer only; edits never move this (TP-044)
  updatedAt                             // changes on every write, including edits
  setBy: 'player' | 'admin'             // 'admin' sets setByUid too; 'player' clears it (TP-077,
  setByUid                              // mirrors sessionEnrollment's choiceSetBy/choiceSetByUid, TP-074)
  // Admin-write only in firestore.rules — every write goes through a server
  // callable (saveAvailability / answerAvailabilityByToken), never a direct
  // client write, so respondedAt/the Wednesday lock can't be bypassed (TP-044).
  // saveAvailability also accepts an optional playerId for an admin setting
  // it on someone else's behalf, skipping the player's own answer-window
  // check (TP-077).

availabilityTokens/{tokenHash}
  weekId, playerId, answer: 'yes' | 'no', expiresAt, testOnly
  // Per-player, per-week one-tap email-answer tokens, stored HASHED. Cloud
  // Functions only — never client-readable or writable, like signInCodes.
  // The token itself is never stored, only its hash. Expires at the week's
  // draftBuiltAt (Wed 9 AM); resolved by getAvailabilityByToken /
  // answerAvailabilityByToken, which never records an answer until the
  // public /answer page's explicit Confirm tap (TP-044).

leagueSettings/main                     // single doc; league-wide values (season-wide, no per-session override)
  golfSimUrl, golfSimLabel              // golf-sim booking link + its button label
  contactName, contactPhone             // "League contact" — powers every "Text Tom" link (TP-057);
                                         // 10-digit phone, same bare-digits convention as players.phone
  weatherVenue: { name, lat, lng } | null // Home weather card's forecast location (TP-060); name 1-40
                                         // chars, lat/lng rounded to 4 decimals; no default — the card
                                         // stays hidden until this is set
  // Menu itself is a PDF in Firebase Storage (league/menu.pdf), not a field here — see Storage below.
  // All writes go through the updateLeagueSettings callable (Admin SDK) (TP-053).

leagueInfo/main
  sections: [{ id, title, body }]        // body is plain text: blank line = paragraph, "- " = bullet,
                                          // **bold** = bold; parsed client-side, never raw HTML
  updatedAt, updatedBy, updatedByName
  // Single admin-editable doc for the public "Rules & league info" page
  // (/info). Public read (including signed out); admin write only, with
  // shape validation in firestore.rules. Never put personal contact details
  // in this doc — a content rule for Tom, not something rules enforce (TP-042).

socialPlans/{weekId}_{playerId}
  weekId, playerId, dinner: 'none'|'before'|'after', golfSim: 'none'|'before'|'after', updatedAt
  // Separate from availability because staff can't read availability (TP-025).

matchGroups/{weekId}_G{groupNumber}
  weekId, slotId, groupNumber (1–6), court
  players: [playerId × 4]               // sorted by Elo descending at pairing time
  setLineups: [{ team1: [p, p], team2: [p, p] } × 3]   // from the fixed rotation; never player-writable
  sets: [{ team1Score, team2Score, savedBy, savedAt } × 3]   // savedBy/savedAt set once a set is saved;
                                                              // savedAt uses Timestamp.now(), never
                                                              // serverTimestamp() (not allowed in an
                                                              // array element). Aligned by index with
                                                              // setLineups; NOT necessarily saved in
                                                              // order (TP-045)
  eloSpread, flagged
  status: scheduled | in_progress | reported | locked | cancelled
  cancelledAt, cancelledBy, cancelReason: string | null, statusBeforeCancel  // only when status is
    // 'cancelled' (TP-061) — a per-slot cancel after pairings are sent (matches_set/in_progress),
    // until the week locks. statusBeforeCancel is what restoreSlot puts it back to. A cancelled
    // group's already-SAVED sets still count at lock (TP-062); its unsaved sets are simply dropped
    // from `emptySets`, not treated as missing. Lock leaves a cancelled group cancelled forever (never
    // locked), so pairing history keeps excluding it even after the week is complete.
  // Removed vs. the old shape: team1, team2, matchNumber (a group can mix one
  // player's first match with another's second). (TP-017)
  // Any of the group's 4 players may write `sets` only, only while status is
  // in_progress/reported (never locked, never cancelled); the admin may write
  // any field, anytime (used to fix a match's scores from the Scores and lock
  // screen, or to cancel/restore a slot).
  // `reported` is never actually written by any code path today — vestigial.

pairingRuns/{weekId}                    // admin-only engine report
  unplaced: [{ playerId, reason: overflow | no_volunteer_fill | no_feasible_slot }]
  twoMatchPlayers[], slotTallies: { playerId: { before, after } }, defaultEloUsed[]
  stats: { maxSpread, totalSpread, flaggedCount, phase1MaxSpread, phase1TotalSpread }
  runAt, runBy

changeRequests/{requestId}
  weekId, matchGroupId, playerId
  type: swap_out | time_change
  requestedSlots: []                    // time_change only — every slot id the player said would work (TP-049)
  fromSlotId                            // the slot they're currently in
  reason, status: pending | approved | denied | withdrawn   // withdrawn = player cancelled while window open
  requestedAt, resolvedAt, resolvedBy
  // Created by a player via submitChangeRequest (while the week is matches_set,
  // i.e. the request window TP-049 defines), OR automatically (type swap_out)
  // when a player opts out of a session mid-way through a week that's already
  // matches_set/in_progress (TP-027, kept in sync with this shape). Approval
  // is recorded only inside the same transaction as the admin's edit
  // (handleEditPairings) that actually performs the swap/replace (TP-049).
  // Admin UI: /admin/changes + a dashboard tile (TP-050).

joinRequests/{requestId}
  name, email, phone, note, status: pending | approved | denied
  createdAt, resolvedAt, playerId (set on approval)
  linkedToPlayerId                      // set instead of a new playerId when approved onto an
                                         // existing player (resolveJoinRequest's linkToPlayerId, TP-075)
  mergedIntoPlayerId                    // set by mergeDuplicatePlayer if this request's own player
                                         // later turned out to be a duplicate and got merged away (TP-075)
  // Public, signed-out submission (submitJoinRequest, no auth required) from
  // someone not yet on the roster; returns {status:'on_roster'} instead of
  // creating a request when the email already matches an existing player.
  // Admin approves (creates a new player + sends an invite, OR links onto
  // an existing one) or denies (resolveJoinRequest). Rate-limited per email
  // per day. Reached from the public /join page (JoinRequestForm) and from
  // Info's "Request to join" call-outs (TP-042).

signInCodes/{sha256(email)}
  playerId, codeHash, salt, expiresAt, attempts, sendTimes[]
  // The emailed 6-digit sign-in code (TP-031). Cloud Functions only — never
  // client-readable or writable. The code itself is never stored, only its
  // salted hash.

eloHistory/{weekId}_{playerId}
  weekId, playerId, seasonId
  eloBefore, eloAfter, delta            // delta is the week's SINGLE summed adjustment (TP-046)
  setsPlayed, setsWon, gamesFor, gamesAgainst
  sets: [{ groupId, setIndex, partnerId, opponentIds, myScore, oppScore, won, expected, adjustment }]
  lockedAt
  // Written once, inside the same transaction as the lock (TP-048); deleted
  // (and players.currentElo restored to eloBefore) on unlock. Never
  // client-writable — see the rules table below. A player with zero sets
  // actually played that week gets no eloHistory doc at all, even if they're
  // listed on a (cancelled or otherwise empty) group (TP-062).

automationHealth/main                   // single doc, automationWatchdog's own health state
  lastCrashAlertAt                      // Timestamp.fromMillis(nowMs), NOT FieldValue.serverTimestamp() —
                                         // the cooldown read compares against the same injected clock
                                         // the write used (TP-066; fixed after a failing test, TPI-037)
  // Cloud Functions only (Admin SDK). Collection name is still a local
  // constant (AUTOMATION_HEALTH_COLLECTION) in functions/automation/watchdog.js,
  // not yet promoted to functions/paths.js — small cleanup candidate.
```

No locked-partner/couples constraint (TP-008).
