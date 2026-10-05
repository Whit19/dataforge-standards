# TNPL — Technical Architecture

**Last updated:** 2026-10-05

## Stack
- Frontend: React + Vite, PWA (installable Home Screen app on iPhone/Android)
- Backend / data: Firebase — Firestore (data), Firebase Auth (emailed 6-digit code + Google), Firebase Storage (menu PDF), Firebase Hosting (deploy), Cloud Functions (43 functions — pairing, season setup, roster/invites/reminders, duplicate-player merge, auth, weekly availability, Lock Week/Elo, weekly automation + its watchdog, cancel a night/slot, change requests, settings, directory sync)
- Firebase project: `tnpl-pwa` (console: https://console.firebase.google.com/u/1/project/tnpl-pwa/overview), Blaze plan (TP-033)
- Repo: https://github.com/Whit19/TNPL (private)
- Local folder: `C:\Dev_Projects\TNPL`
- Docs: `C:\Dev_Projects\dataforge-standards\TNPL\` (cloned locally)
- CLI config: `firebase.json` (firestore rules/indexes, storage rules, `functions` source, hosting rewrites/headers, emulators block incl. storage on port 9199), `.firebaserc`, `firestore.indexes.json`, `storage.rules`
- **Deployed:** Firestore rules, Storage rules, all 43 Cloud Functions, and Hosting are live. Hosting serves `dist/` (built by Vite) plus a `/email/` folder of PNGs used by the invite email, and rewrites `/decline` to the `decline` HTTP function.
- **Standing process rule (TP-067):** Claude Code commits directly on `main`; it does not create a feature branch unless there's a specific, stated reason to.

## Scope for v1
Full weekly loop: availability collection → pairing → live scoring → Elo recalculation. (TP-004)

## League structure
Two seasons per year (e.g. Oct–Dec, Jan–Mar — "season"/"session" used interchangeably). Full player roster is persistent across seasons; opting in to a given season is separate from being on the roster. (TP-009)

## Time-slot / court structure
3 fixed time slots per Thursday, 2 groups (courts) per slot, 4 players per group.
- 6:00 PM — Group 1, Group 2
- 7:15 PM — Group 3, Group 4
- 8:30 PM — Group 5, Group 6

`courtsAvailable` stored per slot per week so this can flex. Group numbers are fixed per court (slot 1 → 1–2, slot 2 → 3–4, slot 3 → 5–6); an unused slot skips its numbers.

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
                                         // the weekly-automation emails, not this one
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
  weekId, playerId                      // ISS-005: written on every doc; the pairing callable can
                                         // still find a week's docs by document-ID prefix "{weekId}_"
  canPlay, canPlayTwo, blockedSlotIds[], preferredSlotIds[], notes
  extraAnswer: boolean | null           // answer to that week's weeks.extraQuestion, if any (TP-055);
                                         // only asked of/counted for players who said they're playing
  respondedAt                           // set on the FIRST answer only; edits never move this (TP-044)
  updatedAt                             // changes on every write, including edits
  // Admin-write only in firestore.rules — every write goes through a server
  // callable (saveAvailability / answerAvailabilityByToken), never a direct
  // client write, so respondedAt/the Wednesday lock can't be bypassed (TP-044).

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
                                         // the write used (TP-066; fixed after a failing test, ISS-037)
  // Cloud Functions only (Admin SDK). Collection name is still a local
  // constant (AUTOMATION_HEALTH_COLLECTION) in functions/automation/watchdog.js,
  // not yet promoted to functions/paths.js — small cleanup candidate.
```

No locked-partner/couples constraint (TP-008).

## Auth & roles — implemented
- Firebase Auth, two sign-in methods, both server-linked to `players` via `playerLinks`:
  - **Emailed 6-digit code (primary)** — `requestSignInCode` / `verifySignInCode` (`functions/auth/signInCode.js`). Primary because an iPhone tapping a link always opens Safari rather than the installed Home Screen app, and a typed code works from inside the installed app either way (TP-031). Code expires in 10 minutes, rate-limited per email per hour, salted-hash stored in `signInCodes` — the code itself is never stored.
  - **Google** — `signInWithPopup` with `select_account` so it always shows the account chooser (fixes a bug where a single-Google-account browser silently reused the last account); `linkAccount` callable does the server-side match/link for a Google-authenticated user with no `playerLinks` doc yet.
- Either path calls `linkUidToPlayer`: writes `playerLinks/{authUid} -> playerId`, sets `players.inviteStatus = 'accepted'`, sets `players.authUid` if not already set, and sets `players.firstSignInAt` once, on the first call only.
- No match by email → user is signed back out, `not_on_roster` state returned; a public join-request form (`submitJoinRequest`) lets them ask to be added.
- `role: 'staff'` — club workers with schedule visibility only, no Elo/pairing/availability/season enrollment.
- `isAdmin: true` — single admin (Tom) in v1.

## Firestore rules — implemented and deployed
`firestore.rules` covers every collection below. Helpers `isSignedInPlayer()` / `isAdmin()` / `isPlayerRole()` centralize the `playerLinks` lookup rather than repeating it per rule.

| Collection | Read | Write |
|---|---|---|
| `players` | **admin + own doc only** (locked down this session, TP-057 — every non-admin screen moved to `directory`/`playerRatings` first) | admin only, except a player may edit their own `phone`, `contactPreference`, `hidePhone`, or `hideEmail` (TP-016, TP-056); `authUid`/`inviteStatus`/`firstSignInAt` are Cloud Function only |
| `directory` | any signed-in member | never client-writable — `directorySync.js` triggers only, on `players`/`sessionEnrollment` writes (TP-056) |
| `playerRatings` | any signed-in member, staff included (reverses TP-016; now `isSignedInPlayer()`, TP-063) | never client-writable — same triggers as `directory` (TP-056) |
| `playerLinks` | own doc only | never client-writable — Cloud Functions only (`verifySignInCode` / `linkAccount`) |
| `seasons` | any signed-in member | **never client-writable** (`allow write: if false`) — `setupSeason` / Settings callables (Admin SDK) only; no client write path depended on direct access (TP-054) |
| `sessions` | any signed-in member | admin only |
| `sessionEnrollment` | admin, or the player themselves (own doc, by `playerId` field) | never client-writable — the `setSeasonSignup` callable only, so every session doc for a player stays consistent |
| `weeks` | any signed-in member (staff: read-only) | **never client-writable** (`allow write: if false`) — `setupSeason` / the pairing engine / Settings callables (Admin SDK) only (TP-054) |
| `leagueSettings` | any signed-in member | never client-writable — the `updateLeagueSettings` callable only (TP-053) |
| `availability` | the player themselves + admin (not staff) | **admin only** — players write through the `saveAvailability` / `answerAvailabilityByToken` callables (Admin SDK), never a direct client write (TP-044) |
| `socialPlans` | any signed-in member, staff included | the player themselves or admin, only while the week isn't `complete`; `dinner`/`golfSim` must be none/before/after; delete admin only |
| `matchGroups` | admin always; everyone else only once the week is `matches_set` / `in_progress` / `complete` (keeps `pairing_draft` groups admin-only). Staff read-only | admin: any field, any status (used to fix a match's scores from the Scores and lock screen, or to cancel/restore a slot); a player: `sets` field only, only while `status` is `in_progress` or `reported` (never `locked`, never `cancelled`), only for a group they're in; `setLineups` etc. otherwise admin / pairing Cloud Function only. No rules change was needed for per-slot cancel — a `cancelled` group was already outside the player-writable status list, and `ScoreEntry.jsx` already treats it read-only (TP-061) |
| `pairingRuns` | admin only | never client-writable — Cloud Function (Admin SDK) only |
| `changeRequests` | the requesting player + admin | never client-writable — `submitChangeRequest`/`withdrawChangeRequest`/`denyChangeRequest` callables, and approval written inside the admin edit-save transaction (`handleEditPairings`), all Admin SDK (TP-049, TP-050) |
| `eloHistory` | any signed-in member, staff included (reverses TP-016; now `isSignedInPlayer()`, TP-063) | never client-writable — `lockWeekInternal` / `unlockWeekInternal` (Admin SDK, inside a transaction) only (TP-048) |
| `joinRequests` | admin only | never client-writable — `submitJoinRequest` / `resolveJoinRequest` (Admin SDK) only |
| `leagueInfo` | public — signed out too (TP-042) | admin only, with shape validation (`sections` list, `updatedAt`/`updatedBy`/`updatedByName`) |
| `availabilityTokens` | nobody | nobody — Cloud Functions (Admin SDK) only |
| `signInCodes` | nobody | nobody — Cloud Functions (Admin SDK) only |

## Elo formulas (ported from the existing Excel system) — implemented and deployed
Pure module `functions/elo/elo.js` (CommonJS, no Firestore access — same style as `pairing/engine.js`), `computeWeekElo({ groups, startElo, kFactor, defaultElo })`:
- K-factor: 32 (editable per season, `season.kFactor`)
- Team Elo = average of the 2 players' individual Elos
- Expected score (Team A) = 1 / (1 + 10^((TeamB_Elo − TeamA_Elo) / 400))
- Margin multiplier = 0.6 + 0.16 × (|point_diff| − 1) — 6-5 → 0.6, 6-3 → 0.92, 6-0 → 1.4
- Per-set adjustment = K × margin_multiplier × (actual_result − expected_score), same adjustment applied to both teammates, negated for the other team. **Each set's teams come from that set's `setLineups` entry** (partners rotate every set), not from a fixed group pairing. An unsaved set is skipped and listed in `emptySets`.
- **Same-week rule (TP-046):** a player's Elo is fixed for the whole week — every set they play (all 3 sets per match, both matches for a two-match player, so up to 6 sets) is computed against the same **start-of-week** Elo, and the deltas are summed into one `delta` per player, applied once at lock.
- Verified against the Excel workbook's worked example: 1600+1500 beat 1450+1400, 6-3 → about ±9.7 (test tolerance ±0.1); zero-sum property (a week's deltas sum to ~0) also tested directly.
- Next-season regression: shrinkage = 0.7 × sets_played / (sets_played + 20); next_start = 1500 + shrinkage × (final_elo − 1500)

## Lock Week and Elo — implemented and deployed
`functions/elo/lockWeek.js`:
- **`previewWeek(db, weekId)`** — a plain read: loads the week/season/groups/players' `currentElo`, runs `computeWeekElo`, and returns `{ results, emptySets (with slot label/court/set number for display), groupCount, canLock, reason, kFactor }`. `canLock` requires the week to be `in_progress` with at least one saved set (locking with some sets still empty is allowed — those sets just don't count).
- **`lockWeekInternal(db, weekId, { by, mode })`** and **`unlockWeekInternal(db, weekId, { by })`** each run inside a single Firestore **transaction**, not a re-read + batch (TP-048): every doc the write depends on (the week, the season, every group, every player's `currentElo`, and — for unlock — every `eloHistory` doc and the season's other weeks) is read via `tx.get()`, so a concurrent lock/unlock attempt that changed any of those between this transaction's read and its commit causes Firestore to abort and retry the whole callback. A write-count guard throws a clear error above 450 writes (a week has at most ~30 players / 6 groups).
  - Lock writes: one `eloHistory/{weekId}_{playerId}` doc per player **who actually played at least one set that week** (full per-set breakdown) — a player with `setsPlayed === 0` (e.g. every group they were in was cancelled or stayed empty) gets no doc and no `currentElo` write at all, rather than a zero-delta no-op (TP-062, fixed a real bug found via the weekly-loop rehearsal, ISS-031); `players.currentElo = eloAfter`, every group's `status` → `locked`, the week's `status` → `complete` plus `lockedAt`/`lockedBy`/`lockMode`.
  - Unlock (`unlockCheck`/`checkUnlock`, TP-047): allowed only for the **most recent** locked week, only while the next week's status is still `draft`/`availability_open`; **aborts the whole unlock, writing nothing,** if any player's live `currentElo` no longer matches what that lock recorded (within 1e-6) — evidence Elo has moved since (e.g. a later week was locked). On success: `eloHistory` docs deleted, `currentElo` restored to `eloBefore`, groups → `in_progress`, week → `in_progress` with `unlockedAt`/`unlockedBy`.
- Admin callables (`functions/elo/callables.js`, admin-only, need Cloud Run public access): **`previewLockWeek`** (names + sorted-by-delta players; also returns `unlockCheck` once the week is `complete`), **`lockWeek`** (returns `{ updated, biggestGain, biggestDrop }`), **`unlockWeek`**.
- `weeklyAutomation` gained two steps (`functions/automation/weekly.js`): a **Tuesday 9 AM** reminder email to admins for any `in_progress` week not yet locked (`lockReminderSentAt` guard), and — immediately **before** building week N's Wednesday draft — an auto-lock of the most recent earlier `in_progress` week if every set is saved (mode `'auto'`), else an email listing the empty sets and the draft still builds on the older Elo.
- Admin screen `src/pages/admin/ScoresLock.jsx` (route `/admin/scores`, tile on the Admin dashboard after Availability): per-match score tables (`GroupScoreTable`, shared with Matches), an "Elo if you lock now" preview (top/bottom 3 + "see all"), inline (non-`window.confirm`) Lock/Unlock confirmations, and an "Elo changes" card reading `eloHistory` once locked. The admin can also fix any match's scores via the normal score page (`ScoreEntry.jsx`, `isAdmin` treated like a group participant while the group is editable), reached from here with router state `{ from: 'admin-scores' }` so its back link returns to this screen. A cancelled match's heading is struck through and muted on this screen so a cancelled slot reads clearly as cancelled rather than just empty.
- Locked "Your match" cards on the player Matches page show the week's Elo change once (`eloHistory/{weekId}_{playerId}`) — on the last match card, worded "Week Elo: …" for a two-match player, since their change is one combined number.

## Scoring & lock flow
- `matchGroups.status`: `scheduled` → `in_progress` → `reported` (editable) → `locked` (immutable)
- Any of the group's 4 players writes `sets` while `in_progress`/`reported` — last write wins (TP-011); the admin can also write any field, anytime, to fix a match from the Scores and lock screen.
- Sets are **not** required to be saved in order (TP-045) — a per-set Firestore transaction replaces only that array index, so two players saving different sets at the same time can't overwrite each other.
- **Lock Week** — see "Lock Week and Elo" above. (TP-012)

## Cancel a night or slot — implemented and deployed (TP-061, TP-062)
- Per-slot cancel, available only after pairings are sent (`matches_set`/`in_progress`), not before: admin callables `cancelSlots`/`restoreSlot` (`functions/cancel/cancel.js`, admin-only, Cloud Run public invoker access set) set/clear `matchGroups.status: 'cancelled'` plus `cancelledAt`/`cancelledBy`/`cancelReason` on every group in the chosen slot(s), each inside its own transaction. A cancelled group is excluded from scoring (rules already block player writes to a `cancelled` group — no rules change needed, see the rules table), from `lockWeekInternal`'s Elo pass (its players, if otherwise unplayed, get no `eloHistory` doc — see "Lock Week and Elo"), and from the pairing engine's prior slot/partner history (`loadPriorHistory` skips `cancelled` groups so a cancelled week never biases future pairing costs).
- Emails go out to every affected player on cancel and on restore (restore is only offered for a group still `in_progress`, i.e. no scores saved and not locked).
- UI: `src/components/CancelBanner.jsx` (Home + Matches, shown instead of the normal match card for a cancelled group/slot, with the reason if given) and an admin screen `src/pages/admin/CancelWeek.jsx` (route `/admin/cancel`) to pick slot(s) and an optional reason, or restore.
- **Whole-week cancel before pairings are sent** is reserved but not built — `WEEK_STATUS.CANCELLED` exists in `src/constants.js`/`functions/paths.js` but nothing writes it yet (deferred; see ProjectRoadmap.md Phase 2).

## Pairing engine — implemented and deployed
Pure module `functions/pairing/engine.js` (CommonJS, no Firestore access, deterministic — all tie-breaks by playerId / slot order) plus the admin-only callable `generatePairings({ weekId })` in `functions/pairing/callable.js`, wired in `functions/index.js` (region us-central1, the default). Design: TP-017 through TP-023, TP-028.

Engine steps:
1. **Overflow** — if more available players than capacity (courts × 4, normally 24), keep the earliest responders; the rest are `unplaced: overflow`.
2. **Fill** — if the count isn't a multiple of 4, add second-match entries from `canPlayTwo` volunteers (best combination by resulting spread); if not enough volunteers, bench the latest responders (`no_volunteer_fill`). A back-to-back preference for two-match volunteers is a soft cost (`TWO_MATCH_GAP_PENALTY = 2.1`), TP-028.
3. **Phase 1** — tightest Elo groups (sorted-Elo chunks of 4), repaired by swaps for hard constraints only: no duplicate player in a group; groups fit slots (no blocked slot, per-slot court capacity, a 2-match player's two groups in different slots). A player who can't be placed anywhere is `no_feasible_slot`.
4. **Slot assignment** — enumerates every valid group→slot arrangement; cost per player = times already played that slot this season (from prior published weeks' `matchGroups`), a preferred slot costs `PREFERRED_SLOT_BONUS = -1`; plus repeat-grouping penalties (same-week repeat 1000, prior-season pairing 0.5 each).
5. **Phase 2** — swap entries between groups to lower total fairness cost, accepted only if hard constraints hold, max and total spread stay within `slotFairnessTolerance` of Phase 1, and no group exceeds `max(phase1MaxSpread, pairingFlagThreshold)`.
6. **Output** — groups (players Elo-desc, `setLineups` from `SET_ROTATION`, `eloSpread`, `flagged` = spread > threshold), unplaced, twoMatchPlayers, slotTallies, stats.

Constants: `functions/paths.js` is the single source of collection names and status strings inside `functions/`; a unit test asserts it (and `SET_ROTATION`) matches `src/constants.js`. Every callable/HTTP function needs Cloud Run public invoker access set manually after deploy (see BestMethods.md).

`generatePairings` callable: requires an admin; week must exist and be `availability_open` or `pairing_draft`; loads availability, enrollment, players, and slot/partner history from earlier published weeks; one batch replaces any prior draft groups, writes the new groups + `pairingRuns/{weekId}`, and sets the week to `pairing_draft`. A missing `respondedAt` counts as the latest responder.

## Publish, admin edits, and weekly automation — implemented and deployed
- `editPairings` (admin) — swap players, move a player, bench a player, replace a benched slot, move or create a match, undo the last edit; every edit is dry-run-previewable before it writes.
- `publishPairings` — moves a week `pairing_draft` → `matches_set` and emails players their match; refuses if any group is short a player (never sends a non-foursome).
- `setPairingHold` — admin can hold a week so the Wednesday auto-send skips it.
- `finalizePairings` — moves a week `matches_set` → `in_progress` (Thursday morning); same short-match refusal.
- `notifyPairingChanges` — emails only the players affected by an admin edit made after publish.
- `weeklyAutomation` — a scheduled function checking every 15 minutes (America/Chicago) against each week's stored `automation` timestamps: builds the draft Wednesday 9 AM, sends Wednesday 5 PM unless held, finalizes Thursday 8 AM (TP-030). Automation times are computed once, at season setup, from each week's Thursday date. It also runs the Tuesday lock-reminder and Wednesday pre-draft auto-lock steps (see "Lock Week and Elo").

## Weekly availability — implemented and deployed
`functions/availability/` — Monday email + Tuesday reminder via `weeklyAutomation`, using each week's `automation.availabilityOpensAt`/`reminderAt`; `availabilityEmailSentAt`/`reminderSentAt` guard against a double-send.
- **One-tap email answers:** per-player, per-week tokens, hashed in `availabilityTokens` (Cloud Functions only). Links go to the public `/answer` page, which **never records an answer on load** — only on an explicit Confirm tap, since email security scanners open every link in an inbound email. A token expires at the week's `draftBuiltAt`; a "Changed your mind?" link on the confirmed page still works until then.
- **In-app form** (`src/pages/Availability.jsx`, route `/availability`): can play, can play 2, Can't/OK/Prefer per slot, dinner/golf-sim plan, notes. No separate confirmation screen — the player returns to Home's availability card, which shows a one-time "Saved" line and their answers. Editable only while the week is `availability_open` and before `draftBuiltAt` (Wed 9 AM).
- Callables (`functions/availability/callables.js`, admin-only where noted): `getAvailabilityByToken`/`answerAvailabilityByToken` (public, signed out), `saveAvailability` (the signed-in player, for themselves), `sendAvailabilityNow` (admin: manual early Monday send / reminder to selected players / a test send to just the admin).
- Admin tracker `src/pages/admin/AvailabilityTracker.jsx` (route `/admin/availability`, tile on the Admin dashboard): counts and filters, reminder to checked players, "Text selected" group text (`sms:` link), "Open availability now" (marks the week's Monday send as done, so the automation skips it — see BestMethods.md), "Send a test to me."

## Player Matches and live score entry — implemented and deployed
- `src/pages/Matches.jsx` (route `/matches`): "Your match" card (per match, 4 states — not started / N-of-3-in / all-in / locked), a Full schedule list (one `GroupScoreTable` per match — a per-player S1/S2/S3/total table, no court numbers, TP-045), and the shared dinner/golf-sim editor.
- `src/pages/ScoreEntry.jsx` (route `/matches/:matchGroupId/scores`): first to 6, no win by 2. **Any set can be entered in any order** (TP-045) — only one set is open for editing at a time, auto-opened only if exactly one set is unsaved. A per-set Firestore **transaction** replaces only that array index (`sets[i] = { team1Score, team2Score, savedBy, savedAt: Timestamp.now() }` — `serverTimestamp()` isn't allowed inside an array element), so two players saving different sets at once can't overwrite each other. Any of the group's 4 players, or the admin (any match, `{ from: 'admin-scores' }` router state), can edit while the group is `in_progress`/`reported`; everyone else sees the same page read-only, with "Live"/"Offline"/"Locked"/"Opens {weekday} {time}" status.
- Shared data hook `src/hooks/useWeekMatches.js` — one source for a week's groups, `bySlot` (via `src/lib/matchSchedule.js`'s `groupsBySlot`, so court number — position within the slot — is computed exactly once and never drifts between pages, ISS-017), the viewer's own group(s), and names.
- Pure score helpers `src/lib/scores.js` (`groupProgress`, `scheduleScoreText`, `teamLabel`, `myMatchResult`, `playerGameRows`, `formatEloDelta`) and shared presentational components `src/components/MatchScores.jsx` (`SetResultList`, `SetTiles`, `GroupScoreTable`) — used by both Matches and Home so a match never renders two different ways.
- `src/lib/socialTimes.js` computes dinner/golf-sim clock times from the published schedule (TP-024): before = first slot start − `dinnerLeadMinutes` (dinner) or − that slot's length (golf sim); after = last slot start + that slot's length; times rounded down to 15 minutes.
- `src/pages/Home.jsx` shows one card per match (never combined for a two-match player, TP-045), set tiles once any set is saved, and a dinner/golf-sim card using the shared `src/components/SocialPlanEditor.jsx` (same editor as Matches).
- "Request a change" now opens the real change-request flow (see "Change requests" below) while the window is open; outside it, or if no League contact is set, it falls back to a text-the-League-contact link or a plain note (TP-043 superseded by TP-049/TP-057).

## Season setup — implemented and deployed
`setupSeason` (admin callable, `functions/season/setupSeason.js`) creates a season, its sessions, and every Thursday week between each session's start/end date (skipping any given skip dates, e.g. Thanksgiving) in one batch. Idempotent — re-running it only touches weeks still in `draft`. Season 26-27 was created this way: Session 1 Oct 15 - Dec 17, Session 2 Jan 7 - Mar 11, Thanksgiving skipped, 19 weeks (TP-029).

## Roster, invites, and join requests — implemented and deployed
`functions/roster/roster.js`:
- `upsertPlayer` (admin) — create or edit a player; starting Elo can't change once they have a locked match. Already rejected two players sharing an email (`assertEmailAvailable`) before the merge work below needed that guarantee.
- `sendInvites` (admin) — `mode` missing or `'invite'` (default): emails the install-first invite (see below) to selected players or every not-yet-invited player; flips `inviteStatus` to `invited` only for players not already signed in (TP-038) — a re-send to a signed-in player still emails them but never touches their status. The Roster screen's select mode can now call it with just the checked players ("Invite selected"), reusing the existing callable rather than a new one (ISS-035, TP-069) — used for the Oct 3 invite send, two days ahead of the planned Oct 5, instead of un-holding "Invite all" (TP-039, still deliberately held). A single-player re-invite from the edit sheet may also target a declined player who has never signed in (TP-071); the bulk paths still skip decliners.
- `sendInvites` `mode: 'reminder'` (TP-073) — a softer nudge for Roster's Pending tab, requires an explicit `playerIds` array. Pure `resolveReminderTargets(players, choiceByPlayerId)` sorts each into a variant or a skip reason: skips `inactive`/`staff`/`declined`/`not_invited`/`has_season_choice` (the last only for a **signed-in** player with a recorded choice — see the known gap below, ISS-043); everyone left is `not_signed_in` or `no_season_pick`. Never touches `inviteStatus`/`invitedAt`/`declinedAt`/the decline token — only a `reminderSentAt` stamp per player on success. `buildReminderEmail({ firstName, variant, appUrl, declineUrl, sessions, email })` builds both variants, reusing `buildInviteEmail`'s own install-steps/sign-in-methods HTML and text (`installAndSignInHtml`/`installAndSignInLines`, extracted into shared helpers so the existing invite-email tests still pass byte-for-byte). The `not_signed_in` variant can't link to the player's actual decline URL — only a salted hash of that token is ever stored, never the plaintext — so it shows a plain "Just reply to this email and let me know" opt-out line instead when no `declineUrl` is given (which is always, on the real send path); `buildReminderEmail` still fully supports rendering a real linked line when one is passed in, for testability. `testToSelf: true` (no `playerIds`) sends both variants to the calling admin's own email, subject prefixed `[TEST] `, no Firestore writes.
- `decline` (HTTP, rewritten from `/decline`) — a player's "stop inviting me" link from the invite email. A GET (and HEAD/OPTIONS) only renders a confirm page and never writes, because email security scanners open every link (ISS-038). Only that page's own POST (`confirmDeclineInvite`) sets `inviteStatus: 'declined'` and `declinedAt` (TP-070). Both responses are the same regardless of whether the token matched, so nothing is revealed to a guesser.
- The player edit sheet's Resend/Send button shows the real outcome inline (TP-072, ISS-040). `inviteSkipReason` in `src/components/admin/PlayerEditSheet.jsx` re-derives why a player was skipped from the same conditions as `resolveInviteTargets` in `functions/roster/roster.js`, because `sendInvites` returns only `sent`/`failed` counts. If that server filter changes, change this function too.
- `submitJoinRequest` (public, signed-out) — someone not on the roster asks to join; rate-limited per email per day; emails every admin.
- `resolveJoinRequest` (admin) — approve (creates a new player + sends an invite, or **links** onto an existing player — see "Duplicate players" below) or deny.
- `setSeasonSignup` (the signed-in player, for themselves, or an admin acting for anyone else — TP-074) — writes the player's `choice` to every session's `sessionEnrollment` doc for the active season in one batch; if opting out drops them from a session whose week is already `matches_set`/`in_progress` and they're in a match, raises a `swap_out` change request instead of silently dropping them (TP-027). An admin targeting someone other than themselves must be admin and the target must be an active player (not staff); the target need never have signed in — `getAvailabilityRecipients`/`loadEligiblePlayers` key only on `optedIn`/`role`/`active`, never `firstSignInAt`, so an admin-set "Full season" for a never-signed-in player really does queue them the Monday email and pairing eligibility once they answer the public one-tap link. Both paths share one write core; only an extra target-eligibility check and the `choiceSetBy`/`choiceSetByUid` stamp differ.

## Duplicate players: merge, and linking a join request onto an existing player (TP-075)
A player who signs in with a different email than the one on the roster hits `not_on_roster` and ends up submitting a join request; approving it the normal way created a **second** `players` doc for the same person, with a guessed starting Elo — the original kept the real Elo/history, the new copy held the sign-in link. ("Brian C Spahn" duplicating "Brian Spahn" was the real example that surfaced this.) Confirmed before building anything: Google sign-in short-circuits on an existing `playerLinks/{uid}` doc before ever re-matching by email, while the emailed-code path always re-resolves through the code's own stored `playerId` (itself set from an email match at send time) — so repointing `playerLinks` plus setting the kept player's email correctly covers both sign-in paths.
- **`mergeDuplicatePlayer({ keepPlayerId, removePlayerId, emailFrom, preview })`** (admin only, new function). `preview: true` returns the plan — both names, the email that would be kept, how many `playerLinks` move, and the season choice that would survive — without writing anything; any refusal reason surfaces the same way for preview and the real merge. Refuses if the ids match, either doc is missing, either is staff, or the duplicate (`removePlayerId`) has any `eloHistory`, is in any `matchGroups`, has an `availability` doc, or has a `changeRequests` doc — game history is never merged, and pre-season this should never be reachable. The real merge is **one transaction**: repoints every `playerLinks` doc pointing at the duplicate, sets the kept player's `email` per `emailFrom` (default `'remove'` — the email the duplicate actually signs in with), adopts the duplicate's `authUid` if the kept player has none, keeps the earlier of the two `firstSignInAt` values, carries the duplicate's season choice onto the kept player only if the kept player has none, deletes the duplicate's `players`/`directory`/`playerRatings`/`sessionEnrollment`/`socialPlans` docs, and stamps the matching `joinRequests` doc's `mergedIntoPlayerId`. `syncPlayerDirectory(db, keepPlayerId)` runs once more right after commit as a deliberate, idempotent belt-and-suspenders call — the `players`/`sessionEnrollment` writes above already re-trigger `directorySync.js`'s own Firestore triggers on their own. That call has to be a **lazy** `require` inside the function, not a top-level one — `directorySync.js` already imports `activeSeason` from this same file, so a top-level require the other way would be circular.
- `resolveJoinRequest`'s approve path now accepts `linkToPlayerId`: instead of creating a new player, it sets that existing player's `email` to the request's email (through `upsertPlayer`, so the existing email-uniqueness check still applies) and sends them the normal invite. Refuses a staff or inactive target, or one who has already signed in (`hasSignedIn`) — that's merge territory instead, since changing just the email field wouldn't repoint an existing `playerLinks` doc.
- UI: a shared pure helper `src/lib/likelySameName.js` (`likelySameName(a, b)`) powers both screens — same last word of the name (ignoring middle initials, punctuation, and a trailing Jr/Sr/II/III/IV) or first/last swapped, case-insensitive. The edit sheet's "Duplicate of another player?" section (collapsed, active non-staff players only) lets Tom pick a likely match or search for one, decides keep vs. remove automatically (whichever of the two came from an approved join request is Remove; otherwise the earlier-created one is Keep, treating a missing `createdAt` as earliest), and previews before merging, with a "Swap" link and an `emailFrom` toggle. Roster's Requests tab shows up to 3 "Already on the roster?" suggestions per request plus an always-available "Link to a different player…" search, and relabels "Approve" to "Approve as new player" once a match exists, so the choice reads as deliberate.

## Invite email — v2, "install-first" (TP-035)
`buildInviteEmail` in `functions/roster/roster.js` builds a table-based, inline-styled HTML email (Gmail/Outlook strip `<style>` and block SVG) plus a plain-text part that mirrors every section. Content, in order: a "not in the App Store" note, a Quick-start box (press-and-hold the button → Open in Safari), a picture of the Home Screen row (PNG, built from the real `apple-touch-icon`, served from `/email/` on hosting), 5 numbered iPhone install steps, separate Android steps, both sign-in methods, the current season's dates (pulled live from `sessions`, not hard-coded), and the existing DataForge footer. Sender display name is "Tom Junker" for this email only (every other email shows "TNPL") — `mailer.js`'s `sendMail` takes an optional `fromName` override for this.

## Email sending (TP-032)
`functions/email/mailer.js` sends through Tom's Gmail account (app password: `GMAIL_USER` / `GMAIL_APP_PASSWORD`); Resend support exists (`RESEND_API_KEY`) but is not enabled. All replies go to Tom's DataForge address; every email gets the same DataForge footer appended in one place.

## iPhone install gate (TP-036)
`src/lib/platform.js`: `isIOS()`, `isStandalone()` (display-mode / `navigator.standalone`), `iosBrowser()` (distinguishes Safari from Chrome/Firefox/Edge/Google-app/in-app browsers on iOS by user-agent token). `SignIn.jsx` renders a "First, add TNPL to your Home Screen" gate instead of the sign-in form whenever `isIOS() && !isStandalone()`, since Safari and the installed Home Screen app keep separate sign-in state. A visible "I'm on a computer" link bypasses the gate for the current browser session (`sessionStorage`, with a try/catch fallback). Desktop and Android are unaffected.

## PWA update-on-resume and build stamp (TP-068, ISS-032)
A real tester's installed iPhone Home Screen app kept showing stale content, since a suspended-then-resumed PWA often never triggers the browser's normal "check the service worker for an update on navigation" path. Fixed with an explicit `registration.update()` call on the `visibilitychange` event, so resuming the app from the background forces a service-worker update check. `src/pages/Home.jsx` also shows a small "Updated {date} {time}" stamp, baked in at build time via Vite's `define: { 'import.meta.env.VITE_BUILD_DATE': ... }` (`vite.config.js`) — giving Tom and testers a quick, no-guesswork way to confirm a given phone actually picked up the latest deploy. The stamp originally showed only a date; a time component was added the same session so two same-day deploys are distinguishable.

## Bottom navigation & Home (TP-037)
`src/components/BottomNav.jsx` — a fixed bottom tab bar (Home, Matches, Rankings, Players, Profile, plus Admin only when `isAdmin`), shown on every signed-in page. `src/pages/ComingSoon.jsx` (the former placeholder for Rankings/Players) is no longer routed to anything now that both are built, but hasn't been deleted yet. `src/pages/Home.jsx` shows an "Are you playing {season}?" card until the signed-in player (not staff) has a season choice on record — gone for good once one exists, with a same-visit-only green confirmation right after choosing — plus a pre-season card (season name, start date, when week-1 availability opens) while today is before the first week.

## Rules & league info, and the public join page (TP-042)
- `src/pages/Info.jsx` (route `/info`, public — readable signed out, outside the iPhone install gate) renders `leagueInfo/main`'s `sections[]` via `src/lib/infoText.js`, a small parser for plain text with light formatting (blank line = paragraph, `- ` = bullet, `**bold**` = bold) — output is React elements, never raw HTML. Shows "Member? Sign in" and a "Request to join" call-out when signed out.
- Admin editor `src/pages/admin/InfoEdit.jsx` (route `/admin/info`) pre-fills starter text (Tom's rules plus a League setup section) the first time the doc doesn't exist.
- `src/pages/Join.jsx` (route `/join`, public, outside the install gate) hosts `src/components/JoinRequestForm.jsx` (extracted from the admin flow so both share it), with an email field; `submitJoinRequest` returns `{status:'on_roster'}` for an existing member instead of creating a request, and the form shows an "already on roster" card for that case.
- Home's top-right initials link was replaced with an "ⓘ Info" pill (icon + word) linking to `/info`; the same link was added to Sign In, the install gate, and the not-on-roster screen.

## Roster status — `firstSignInAt` is the source of truth (TP-038, TP-041)
`src/lib/rosterStatus.js`: `hasSignedIn(player)` is true if `firstSignInAt` is set OR `inviteStatus` is `accepted`. `classify(player, choice)` derives the Admin Roster screen's four tabs from that plus the player's `sessionEnrollment` choice for the active season (`choiceByPlayerId` from `useRoster`):
- **Active** — signed in AND opted into a session (`full`/`session_1`/`session_2`); staff are exempt from the season-choice requirement, since they're never asked.
- **Pending** — invited with no response yet, signed in but no season answer yet, **or a recorded playing choice while still not signed in** (TP-074) — Active always requires both signed-in and opted-in.
- **To Invite** — never invited (a brand-new add, or an approved join request) and no choice recorded.
- **Not Active** — `active: false` (former roster), declined the invite with no recorded choice, or a recorded `none` choice — whether or not they'd ever signed in (TP-074, fixed a gap where `classify` ignored `choice` entirely for a not-signed-in player, so an admin-set "Not this season" for someone who'd never installed the app used to have no effect at all).
- `pendingReason(player, choice)` (TP-073) and `notActiveReason(player, choice)` (TP-074) give the *why* behind Pending/Not Active, each `null` outside its own tab: `pendingReason` is `not_signed_in` or `no_season_pick`; `notActiveReason` is `inactive`, `declined`, or `not_this_season` (the caller still checks `hasSignedIn` itself to pick between "Signed in · Not playing this season" and the not-signed-in wording, since one reason maps to two labels). `reminderSkipReason(player, choice)` (TP-073) is the client-side mirror of `resolveReminderTargets` and must change with it.

The player edit sheet's "This season" section (TP-074) lets an admin record a choice directly: current status line (reusing the same 4 labels as Home's own "Are you playing?" picker, `HOME_SEASON_CARD.choices` — not retyped), a "· set by admin" marker when `choiceSetBy === 'admin'`, an inline confirm before saving, and no option to clear back to "no pick yet". It reads the target player's live role/active/choice via its own `onSnapshot` listeners rather than the sheet's frozen player prop (ISS-041 — scoped to just this section; the rest of the sheet, e.g. the Resend/Send button's label, still reads the frozen prop and remains deferred).

A one-time script, `functions/scripts/backfillSignIns.js` (dry-run by default, `--apply` to write, idempotent), sets `firstSignInAt`/`inviteStatus` for anyone with a `playerLinks` doc who's missing them — needed for accounts linked before this logic existed.

## Change requests — implemented and deployed (TP-006, TP-049, TP-050)
Player flags an issue with their assigned group (swap out / time change), via `src/pages/RequestChange.jsx` (route `/matches/:matchGroupId/request`), or opting out mid-session raises one automatically (TP-027) → `pending` → admin approves/denies. The request window is open from pairings sent (`matches_set`) until finalize (`in_progress`); outside it, the UI falls back to the League-contact text link (TP-057) instead of the request form.
- All writes go through callables (`functions/changeRequests/changeRequests.js`): `submitChangeRequest` (player, `type: 'time_change'` with 1+ `requestedSlots`, or `'swap_out'`), `withdrawChangeRequest` (player, only while `pending` and the window is open), `denyChangeRequest` (admin). `changeRequests` rules are `allow write: if false` — no direct client write.
- **Approval** happens only inside `handleEditPairings`'s own Firestore transaction (`functions/pairing/edit.js`, converted from read-then-batch to a transaction this session) — the change-request doc is read via `tx.get` before any writes, so a request is marked `approved` only if the admin's resulting swap/replace edit actually commits; backing out leaves it `pending` (TP-049).
- Admin UI: `/admin/changes` (`src/pages/admin/ChangeRequests.jsx`) lists pending/resolved requests, a dashboard tile shows the pending count; approving a request reopens `PairingDraft.jsx`'s existing `PlayerSheet` with it preselected rather than a separate swap screen (TP-050).
- Card state-selection logic (pending/approved/denied/withdrawn, closed-window fallback) is shared in `src/lib/changeRequestCard.js`, consumed separately by Home's and Matches' own card components.
- A swapped-out player sees a one-time "You're off this week's schedule" notice on Home for that week. **Not yet built:** per-slot pairing re-run on approval (ISS-006, deferred) — the callable re-runs the whole week only.

## Rankings, History, and Season summary — implemented and deployed (TP-051)
- Pure, tested module `src/lib/standings.js` (`computeStandings`, `sortStandings`, `computePreseason`, `seasonHighlights`, `playerSeasonTimeline`, `shortName`) computes every number shown on these pages from `eloHistory` on the device — there is no stored standings doc.
- `src/pages/Rankings.jsx` (route `/rankings`, plain signed-in guard): season picker (newest first, "· Final" for a `complete` season), sort by Elo / Win % / Avg games / Matches, a Provisional badge (`season.provisionalUnderSets`, fallback 12), rows link to History. Before Week 1 locks, shows players ranked by starting Elo with a "play starts" note (`computePreseason`) — the pre-season list now comes from `directory` (`role === 'player' && listed === true`) rather than a direct `sessionEnrollment` query, since that query wasn't provable under the Firestore rules for a non-admin and threw `permission-denied` (ISS-028, TP-064). **Staff now see the real page**, not a "Rankings are for players" message — the staff-only early return was removed and `usePlayerRatings()` is always enabled, since every row's Elo depends on it, not just a staff viewer's own (TP-063, reverses TP-016).
- `src/pages/History.jsx` (routes `/history`, `/history/:playerId`): season picker, stat tiles, weeks grouped by session with per-match set detail (`playerSeasonTimeline`). `weekIds` passed to the `matchGroups` query is now filtered to `complete` weeks only (`WEEK_STATUS.COMPLETE`) before the query runs, fixing a second `permission-denied` on an active season still entirely in `draft` (ISS-029, TP-065) — with no complete weeks, the listener isn't opened at all and the page falls through to its existing empty state. At bare `/history`, staff default straight to the "All players" view (no "Mine" tab, since staff have no history of their own); at `/history/:playerId` everyone — including a player arriving from a Rankings-row tap — now lands directly on that player's detail view instead of needing to tap "Mine" first (ISS-033); the "View my history" link is hidden for staff as well as for the viewer's own row.
- `src/pages/SeasonSummary.jsx` (route `/seasons/:seasonId/summary`): highlights (Top Elo, Most sets won, Most matches, Best win % at 30+ sets via `seasonHighlights`), full standings, session tabs. The approved mockup's "next season starting Elos" link is intentionally not built — there's no season-close record yet to link to. Staff also see the real page now, same TP-063 reversal — the staff-only early return and its now-unused `isStaff`/`ROLES` import were removed.
- All three read names via the shared `src/hooks/useDirectory.js` hooks (see "Contact privacy" below) rather than reading `players` directly.

## Settings — implemented and deployed (TP-053, TP-054)
Admin screens `src/pages/admin/Settings.jsx` (season/league values) and `src/pages/admin/Season.jsx` (time slots, skip a week), both reading/writing `functions/settings/settings.js`'s 7 callables: `updateSeasonSetting` (range-validated K-factor, flag threshold, slot fairness, new player Elo, guest Elo, provisional-under-sets, carry-over factor/pivot, dinner lead minutes, back-to-back toggle), `updateDefaultTimeSlots`, `updateWeekTimeSlots`, `resetWeekTimeSlots`, `setWeekSkipped`, `updateLeagueSettings`, `setAdmin`.
- **Time slots:** a season `defaultTimeSlots` applies to upcoming `draft` weeks that aren't `customTimes`; a per-week edit/reset is allowed only while `draft`/`availability_open` — moved times keep their slot ids and players' existing answers, a removed slot's id is stripped from that week's `availability` docs. Courts are set per slot (the pairing engine already uses each slot's own `courtsAvailable`).
- **Skip a week:** only while `draft` (new week status `skipped`); later draft weeks renumber (`weekNumber`/`sessionWeekNumber`) via `planWeekRenumber`.
- **Admins:** added from the signed-in roster via `setAdmin`; an admin can't remove themselves, and the league can never be left with zero admins.
- **League contact:** `contactName`/`contactPhone` on `leagueSettings/main`, surfaced in a `LeagueContactSheet` — see "Contact privacy" below.
- **Menu:** an uploaded PDF, not a field — see Firebase Storage below.
- League name/logo and the automation times are read-only in Settings (baked at build time / fixed elsewhere). `carryOverFactor`/`carryOverSetsPivot` are stored but have no consuming logic yet (season close isn't built). `seasons`/`weeks` are now fully server-write only (see the rules table).

## Firebase Storage — implemented and deployed (TP-054)
Enabled on `tnpl-pwa` (region matched to Firestore). `storage.rules` mirrors `firestore.rules`' identity helpers via cross-service `firestore.get()`/`firestore.exists()` (storage rules can only check a known Firestore path, never query, same constraint as Firestore rules themselves). Only path in use: `/league/menu.pdf` — signed-in read; admin-only write/delete, content-type restricted to `application/pdf`, size capped at 10 MB. `firebase.json` gained a storage-rules entry and a storage emulator on port 9199; `src/firebase.js` exports a `getStorage` client. "View menu" fetches the download URL ahead of time and renders a real `<a target="_blank">`, since iOS blocks `window.open` called after an `await` (ISS-021). The link itself is a small reusable `ViewMenuPill` component (exported from `src/components/DinnerGolfLinks.jsx` alongside the default-exported golf-sim-link component, both sharing the new `useLeagueSocialSettings()` hook), placed as a header pill on Home's and Matches' dinner/golf-sim card headers rather than inline text (ISS-036 wording pass).

## Week extras — implemented and deployed (TP-055)
- **Dinner special:** `weeks.dinnerSpecial` (≤120 chars), set via `setWeekDinnerSpecial`, shown with one consistent "This week's special:" label on the availability form, Matches (both states), Home, and in the availability email if set before sending.
- **Extra yes/no question:** `weeks.extraQuestion.text` (≤80 chars), set via `setWeekExtraQuestion`; answers in `availability.extraAnswer`. Editable while `draft`/`availability_open`; rewording keeps existing answers, removing the question deletes that week's answers in one transaction. Only players who said they're playing are asked or counted.
- Players answer on the in-app form, or right after tapping "Yes, I'm in" on the public `/answer` page (never on page load) via the new public callable `answerExtraQuestionByToken` — same no-answer-on-load pattern as the main one-tap tokens (TP-044).
- The Monday email shows a boxed extras block when a dinner special and/or extra question are set. Admin tracker (`src/lib/extraQuestion.js`) shows Yes / No / No answer counts with names; the Admin dashboard has a "This week" card linking both sheets. "Send a test to me" links are test-only and never record an answer, by design.

## Contact privacy — directory & playerRatings — implemented and deployed (TP-056, TP-057)
Firestore rules can't hide individual fields, so contact info for other people now comes only from server-maintained collections — see the `directory`/`playerRatings` schema above and the `functions/players/directorySync.js` trigger module (fires on both `players` writes and `sessionEnrollment` writes, via the shared `syncPlayerDirectory(db, playerId)`, since a session opt-in/out can flip `listed` without a `players` write happening).
- **Listing rule:** a player-role person is listed (visible on the Players page, with contact info) only if `active`, has signed in (`firstSignInAt` set), and has opted into the ACTIVE season (`sessionEnrollment.optedIn === true` for at least one session). Staff are listed as soon as they're `active` — never asked to opt in. Everyone else still gets a name-only `directory` entry (`listed: false`, no phone/email regardless of hide flags) so Rankings/History/match screens can still show a name.
- `src/pages/Players.jsx`: Players/Staff filter, search, Call/Text/Email links where present, "Contact hidden" otherwise, names link to History.
- Six non-admin screens read `directory`/`playerRatings`/`leagueSettings` instead of `players`, through shared hooks in `src/hooks/useDirectory.js`: `useDirectory()`, `usePlayerRatings({ enabled })`, `useLeagueContact()` — consumed by `useWeekMatches` (Home/Matches), `ScoreEntry`, `RequestChange`, `Rankings`, `History`, `SeasonSummary`.
- **League contact** (`leagueSettings.contactName`/`contactPhone`) replaces scanning `players` for `isAdmin` to build "Text Tom" links, independent of anyone's profile privacy; falls back to "Contact the league admin" when unset.
- `players` reads are now admin + own doc only (see the rules table) — locked down only after an audit confirmed every remaining client reader of `players` was admin-only or own-doc.
- One-time `functions/scripts/backfillDirectory.js` (dry-run default, `--apply`) populates `directory`/`playerRatings` for the existing roster; must be re-run after a new season is created, since everyone starts unlisted again until they choose for that season. Deploy order: functions → backfill → League contact set in Settings → hosting → verify every moved screen → **then** the `players` rules lock, last (TP-057).
- `buildDirectoryEntries()` writes `null`, never `undefined`, for `currentElo`/`seasonStartElo` when the source `players` field is missing (TP-058) — found when a change-request "replace with a guest" edit crashed the trigger outright (ISS-027); guests get a `currentElo` but never a `seasonStartElo`.

## Full emulator dress rehearsal — done (TP-059)
One-time test harness `functions/scripts/runWeeklyLoop.js` (never deployed; emulator-only) drives `runWeeklyAutomation(db, now)` through 2 complete weeks plus a 3rd week's Monday/Wednesday handoff, calling every callable in-process (`handleEditPairings`, `submitChangeRequest`/`withdrawChangeRequest`/`denyChangeRequest`, `saveAvailability`, `answerAvailabilityByToken`, `lockWeekInternal`/`unlockWeekInternal`, etc. — all plain `(db, request)`-style exports, not through the Functions-emulator HTTP layer) as different simulated players. `FUNCTIONS_EMULATOR='true'` is set before `firebase-admin` loads so `mailer.js` routes every send to its `console.log` branch — no real email is reachable, and the script captures and counts every one (181 in the final run). 74/74 checks passed: Monday send/no-double-send, token + in-app answers with a no-answer-on-load check, the Tuesday reminder, the Wednesday draft and its placement constraints, the Wednesday send, change requests (withdraw, approve-in-transaction, deny, refused once the window closes), Thursday finalize, out-of-order scores, the Tuesday lock reminder, auto-lock skipped for one empty set (with the admin email and the next draft building on the pre-lock Elo), a manual lock and the unlock refusal, a fully-scored week's auto-lock, and Elo/`playerRatings`/`computeStandings` cross-checks including one set recomputed by hand against the formula. The simulated season is dated December 2026 so two checks that deliberately use the real wall clock (`weekAcceptsAnswers`, a token's `expired` check) don't trip — needs re-dating after mid-December 2026 to keep working.

## Automation watchdog — implemented and deployed (TP-066)
- `functions/automation/watchdog.js`: pure `findOverdueSteps(weeks, nowMs, { graceMinutes = 30 })` checks each active season week's automation steps (availability-open email, draft build, publish send, finalize) independently — not mutually exclusive — against that week's stored `automation` timestamps plus a 30-minute grace period, and returns the overdue ones with a short note and the week it belongs to. `runWatchdog(db, nowMs)` loads the active season's weeks, runs the check, and — only if anything is overdue — emails every admin one combined summary (`buildWatchdogEmail`) via the shared `emailAdmins` helper from `functions/pairing/publish.js`.
- Registered as its own scheduled function, `automationWatchdog` (`every 60 minutes`, `America/Chicago`, in `functions/index.js`), separate from `weeklyAutomation` so a stuck or crashed automation run doesn't also take down its own watchdog. Confirmed via `gcloud scheduler jobs list --project=tnpl-pwa --location=us-central1` that deploying it created a new Cloud Scheduler job automatically, with public invoker access already set the same way as every other scheduled function.
- **Crash alerting:** `weeklyAutomation`'s handler is now wrapped in try/catch; on a throw, `notifyAutomationCrash(db, error, nowMs)` emails every admin (`buildCrashEmail`) and rethrows, so the function still reports failure to Cloud Functions/Scheduler as before — the alert is additive, not a swallow. A `automationHealth/main` doc's `lastCrashAlertAt` enforces a one-hour cooldown so a repeatedly-crashing function doesn't spam admins every 15 minutes; the cooldown write uses `Timestamp.fromMillis(nowMs)`, not `FieldValue.serverTimestamp()`, so the stored value and the read-side comparison are both in terms of the same injected clock (TP-066's own testsuite caught the original `serverTimestamp()` version always re-emailing, since a test fakeDb never resolves that sentinel to a real time — ISS-037).
- `watchdog.test.js` (12 tests) was written under an explicit "tests only, don't bend a failing test to make it pass" rule and caught both the `notifyAutomationCrash` clock bug above and, earlier, a self-caught `findOverdueSteps` logic bug (an OR where the intended condition — matching `weekly.js`'s own gate — was AND) before any test ran.

## Season 25-26 history import — done (TP-052)
One-time script `functions/scripts/importSeasonHistory.js` (dry run default, `--apply`, single Firestore batch, aborts on any mismatch, every set validated against the live `computeWeekElo`) imported `TNPL_Season_25-26_ALL.xlsx`: `seasons/2025-26` (`status: 'complete'`, `imported: true`), 2 closed `sessions`, 17 `complete` weeks (`lockMode: 'import'`), 68 locked `matchGroups`, 251 `eloHistory` docs — `players` was never touched. Validation tolerance is ±0.15 to match the workbook's 1-decimal Elo display vs. its 2-decimal set-change precision; the stored `delta` stays the exact sum of the set adjustments regardless (ISS-022).

## Weather — implemented and deployed (TP-060), Batch C
- `src/lib/weather.js`: every export pure except `fetchForecast`. `buildForecastUrl({lat,lng})` builds the Open-Meteo hourly URL (feels-like, precip probability, snowfall, wind speed/gusts/direction; °F, mph, inch; `forecast_days=7`) — no API key, called straight from the browser, no Cloud Function, no rules change. `slotHourKey`/`compass` are small pure helpers. `layerBadge(feelsLikeF)` rounds to a whole degree first, then applies Tom's rule via two named constants, `LAYER_ONE_ABOVE = 40` and `LAYER_THREE_BELOW = 30` (>40 → "1 layer", 30-40 inclusive → "2 layers", <30 → "3 layers"). `summarizeNight({hourly, dateISO, slots})` returns per-slot feels-like/wind, the **coldest slot's** badge (not the viewer's own slot), a precip window from the first slot's start hour through the last slot's end hour (`precipChance` = max probability in that window, `precipKind` = "snow" only if any forecast snowfall > 0 in that window, else "rain" — snowfall-based, not a feels-like cutoff, for accuracy), and `maxGust` within the same window; returns `null` if any slot's hour is outside the 7-day forecast. `showWeatherForWeek({weekDateISO, weekStatus, nowChicagoISO})` is true only Monday 00:00 through Thursday 23:59 Chicago of that week, never for `status: 'skipped'` — pure calendar-day string arithmetic (same technique as `functions/automation/weekly.js`'s `addDaysISO`), no timezone library needed since the caller resolves "now" to Chicago first. `fetchForecast({lat,lng,now})` caches the raw hourly JSON in `localStorage` for 60 minutes (key by lat/lng); every storage access is wrapped in try/catch (private browsing, or this module's own tests); a failed fetch falls back to a stale cached copy if any, else `null` — never throws.
- `src/components/WeatherCard.jsx`: dumb component, `{venueName, night, mySlotIds}`; renders `null` if `night` is null. One tile per slot (own slot highlighted white/bordered, label "{time} · You"), feels-like in Barlow Semi Condensed, "Wind {dir} {mph}", a per-tile full-word `aria-label`. Footer: "gusts to N" only when `maxGust` exceeds the highest slot's wind by more than 5 mph, plus the precip line.
- `src/pages/Home.jsx`: the week shown is picked directly from the already-subscribed `effectiveWeeks` using `showWeatherForWeek` itself as the picker — **not** `useWeekMatches()`'s `matchWeek`, which only ever resolves once a week is `matches_set`/`in_progress`/`complete` (rules block earlier matchGroups reads), so reusing it would show last week's weather or none at all during `draft`/`availability_open`/`pairing_draft`. Slots come from `week.timeSlots` (already the per-week-resolved list, custom times included) via a small locally-duplicated `slotLengths`-style helper (`src/lib/socialTimes.js`'s own version isn't exported — same "kept in sync by hand" choice as this file's `HomeChangeRequestWithdraw`). Fetched once per mount per venue; the card renders via two call sites (inside the published-week block, between the match cards and the dinner/golf-sim card; and a fallback for every other week state) so it never renders twice and never disappears behind an early return. `mySlotIds` only populates when `matchWeek` happens to be the same week as the weather week (i.e. already published) — otherwise empty, same as for staff and anyone not playing.
- `leagueSettings.weatherVenue` (`{name, lat, lng}` or `null`) is set via `updateLeagueSettings` and a new "Weather location" row in `src/pages/admin/Settings.jsx` (League group, after League contact) — accepts a pasted "lat, lng" pair in the Latitude field. No default coordinates; the card stays hidden until a real venue is set. Its UI text lives in a local `WEATHER_VENUE_TEXT` object in `Settings.jsx` rather than the shared `SETTINGS_TEXT` in `constants.js` (a one-prompt-one-file constraint, not a deliberate pattern) — a candidate for a small later cleanup.
- Checked, not changed: no CSP exists in `firebase.json` or `index.html`, and the service worker (`src/sw.js`) only routes the precache manifest and a navigation-only SPA fallback — neither intercepts the cross-origin `fetch()` to `api.open-meteo.com`.
- Not yet seen rendering in a real browser — traced by hand against the mockup and unit-tested, but the card can't show in production before **Mon Oct 12** (Monday of Season 26-27's first week) regardless of when the venue is set, since `showWeatherForWeek` gates on the real date.

## Pages
Home · Matches (+ score entry, request change) · Rankings · History (+ per-player, + Season summary) · Players · Profile · Availability · Info (public) · Join (public) · AvailabilityAnswer (public) · Admin (dashboard, season setup, Season (time slots/skip), Settings, roster, pairing draft/edit screens, availability tracker, Scores and lock, change requests, Info editor, Cancel) · **SignIn** (route-guard redirect target; also renders the iPhone install gate — not a nav item). Bottom tab bar on every signed-in page (TP-037). Staff see the same tabs as a player, minus the season-signup card on Home — Rankings/History/Season summary are now fully visible to staff too (TP-063, reverses TP-016).

## Local emulator testing
Firebase Local Emulator Suite (auth 9099, firestore 8080, functions 5001, UI) under the demo project ID `demo-tnpl`, so nothing can reach real Firebase resources (TP-026). Needs JDK 21+ (firebase-tools 15.x).
- Start: `firebase emulators:start --only auth,firestore,functions --project demo-tnpl`
- End-to-end pairing check: `node functions/scripts/seedAndRunPairings.js` (repo root) — resets emulator state, seeds `functions/scripts/fixtures/roster.seed.json` (45 players, names + Season 3 Elos only) with synthetic `@example.test` emails, calls the callable as different users, runs 12 checks.
- Full-league-night demo: `node functions/scripts/seedMatchesDemo.js` — Demo Player/Demo Admin accounts; Week 1 complete and **locked the real way** via `lockWeekInternal` (so it has real `eloHistory`), Week 2 `in_progress` with mixed score states left for Tom to lock/unlock himself, Week 3 `availability_open`. Self-checks include eloHistory doc count, Elo deltas summing to ~0, and `previewWeek` on Week 2.
- Unit tests: `npm test` from the repo root (Node's built-in test runner, auto-discovers every `*.test.js`) — **472 total** (392 on Oct 2, up from 340 at the start of 2026-10-01), spanning `functions/` and `src/`. `npm test` inside `functions/` alone still works and runs just its own subset. The weekly-loop rehearsal (`node functions/scripts/runWeeklyLoop.js`) was re-run after the season-choice admin override, since it touches an availability input — still 74/74.
- New files added since Oct 2: `functions/cancel/cancel.js` + `cancel.test.js` (per-slot cancel/restore), `functions/automation/watchdog.js` + `watchdog.test.js` (overdue-step + crash-alert watchdog), `src/components/CancelBanner.jsx`, `src/pages/admin/CancelWeek.jsx` (route `/admin/cancel`), `ViewMenuPill` (exported from the rewritten `src/components/DinnerGolfLinks.jsx`), `src/lib/socialTimes.js`'s `playerIds` addition to each dinner/golf-sim bucket (ISS-036), and `src/lib/likelySameName.js` + its test file (TP-075).
- Full weekly-loop dress rehearsal: `node functions/scripts/runWeeklyLoop.js` — see "Full emulator dress rehearsal" above. Separate from the unit-test suite; not run by `npm test`.
- One-time data scripts (`functions/scripts/`) — `importRoster.js`, `updatePhones.js`, `backfillSignIns.js`, `resetWeekAvailability.js` (refuses if pairings exist for the week; Tom used it against production to reset Week 1 after testing early), `importSeasonHistory.js` (Season 25-26 import, TP-052), `backfillDirectory.js` (contact-privacy directory/playerRatings backfill, TP-057) — all dry-run by default, `--apply` to write, real project or `--emulator`; see BestMethods.md.

## Known constraints / preferences
- Local dev on Windows 11, VS Code, PowerShell.
- Firebase config values (`VITE_FIREBASE_*`) go directly into `.env.local` by Tom — never relayed through chat.
