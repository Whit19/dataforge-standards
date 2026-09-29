# TNPL — Technical Architecture

**Last updated:** 2026-09-28

## Stack
- Frontend: React + Vite, PWA (installable Home Screen app on iPhone/Android)
- Backend / data: Firebase — Firestore (data), Firebase Auth (emailed 6-digit code + Google), Firebase Hosting (deploy), Cloud Functions (17 functions — pairing, season setup, roster/invites, auth, weekly automation; Elo calc / week-lock not yet built)
- Firebase project: `tnpl-pwa` (console: https://console.firebase.google.com/u/1/project/tnpl-pwa/overview), Blaze plan (TP-033)
- Repo: https://github.com/Whit19/TNPL (private)
- Local folder: `C:\Dev_Projects\TNPL`
- Docs: `C:\Dev_Projects\dataforge-standards\TNPL\` (cloned locally)
- CLI config: `firebase.json` (firestore rules/indexes, `functions` source, hosting rewrites/headers, emulators block), `.firebaserc`, `firestore.indexes.json`
- **Deployed:** Firestore rules, all 17 Cloud Functions, and Hosting are live. Hosting serves `dist/` (built by Vite) plus a `/email/` folder of PNGs used by the invite email, and rewrites `/decline` to the `decline` HTTP function.

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
  invitedAt, declineTokenHash           // set only while inviteStatus is 'invited'
  firstSignInAt                         // set once, on the first successful sign-in — the durable
                                         // "has this player signed in" marker (TP-038), independent
                                         // of inviteStatus (which a later re-invite can otherwise touch)
  contactPreference: 'email' | 'push' | 'both'

playerLinks/{authUid}
  playerId
  // Reverse index, auth UID -> playerId. Firestore security rules can only
  // do get() by a known document path, not arbitrary queries — this is what
  // lets isAdmin()/isPlayerRole() rules helpers resolve "which player is the
  // caller" from request.auth.uid alone. Written by the auth Cloud Functions
  // (verifySignInCode / linkAccount) alongside inviteStatus/authUid/
  // firstSignInAt on the matched player doc. (TP-014)

seasons/{seasonId}                      // e.g. '2026-27'
  name, startDate, endDate, status: 'active'
  kFactor, pairingFlagThreshold (TP-018), slotFairnessTolerance (TP-021),
  newPlayerElo, guestElo, provisionalUnderSets, carryOverFactor, carryOverSetsPivot,
  dinnerLeadMinutes, automation: { availabilityOpens, reminder, draftBuilt, pairingsSent, finalized }
  defaultTimeSlots

sessions/{seasonId}-s{number}           // a season has two sessions (TP-029)
  seasonId, number, name: "Session 1", startDate, endDate
  status: not_started | open | closed

sessionEnrollment/{sessionId}_{playerId}
  playerId, sessionId, seasonId
  choice: full | session_1 | session_2 | none    // the player's one answer, copied onto every session doc
  optedIn: boolean                               // derived per session from choice + that session's number
  updatedAt
  // Written only by the setSeasonSignup callable, which keeps every session
  // doc for a player consistent in one batch.

weeks/{weekId}                          // weekId is the ISO date, e.g. '2026-10-15'
  seasonId, sessionId, date, weekNumber, sessionWeekNumber
  status: draft | availability_open | pairing_draft | matches_set | in_progress | complete
  timeSlots: [{ id, label: "6:00 PM", start, courtsAvailable: 2 }, ...]   // slot order = array order
  pairingHold: boolean                  // admin can hold the Wednesday auto-send
  pendingNotify: []                     // players to notify after a post-send admin edit
  automation: { availabilityOpensAt, reminderAt, draftBuiltAt, pairingsSentAt, finalizedAt }  // Timestamps

availability/{weekId}_{playerId}
  canPlay, canPlayTwo, blockedSlotIds[], preferredSlotIds[], notes, respondedAt
  // No weekId/playerId fields yet — the pairing callable finds a week's docs by
  // document-ID prefix "{weekId}_". Plan: write weekId + playerId fields when the
  // availability form is built (the seed script already does; both lookups work).

socialPlans/{weekId}_{playerId}
  weekId, playerId, dinner: 'none'|'before'|'after', golfSim: 'none'|'before'|'after', updatedAt
  // Separate from availability because staff can't read availability (TP-025).

matchGroups/{weekId}_G{groupNumber}
  weekId, slotId, groupNumber (1–6)
  players: [playerId × 4]               // sorted by Elo descending at pairing time
  setLineups: [{ team1: [p, p], team2: [p, p] } × 3]   // from the fixed rotation; never player-writable
  sets: [{ team1Score, team2Score } × 3]               // scores only, aligned by index with setLineups
  eloSpread, flagged
  status: scheduled | in_progress | reported | locked
  // Removed vs. the old shape: team1, team2, matchNumber (a group can mix one
  // player's first match with another's second). (TP-017)

pairingRuns/{weekId}                    // admin-only engine report
  unplaced: [{ playerId, reason: overflow | no_volunteer_fill | no_feasible_slot }]
  twoMatchPlayers[], slotTallies: { playerId: { before, after } }, defaultEloUsed[]
  stats: { maxSpread, totalSpread, flaggedCount, phase1MaxSpread, phase1TotalSpread }
  runAt, runBy

changeRequests/{requestId}
  weekId, matchGroupId, playerId
  type: swap_out | time_change
  reason, status: pending | approved | denied
  requestedAt, resolvedAt, resolvedBy
  // Created by a player flagging a group issue, OR automatically (type
  // swap_out) when a player opts out of a session mid-way through a week
  // that's already matches_set/in_progress (TP-027). Admin approve/deny UI
  // not built yet — requests currently just queue up.

joinRequests/{requestId}
  name, email, phone, note, status: pending | approved | denied
  createdAt, resolvedAt, playerId (set on approval)
  // Public, signed-out submission (submitJoinRequest) from someone not yet on
  // the roster; admin approves (creates the player + sends an invite) or denies
  // (resolveJoinRequest). Rate-limited per email per day.

signInCodes/{sha256(email)}
  playerId, codeHash, salt, expiresAt, attempts, sendTimes[]
  // The emailed 6-digit sign-in code (TP-031). Cloud Functions only — never
  // client-readable or writable. The code itself is never stored, only its
  // salted hash.

eloHistory/{weekId}_{playerId}
  eloBefore, eloAfter, delta
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
| `players` | any signed-in member | admin only, except a player may edit their own `phone` (TP-016) or `contactPreference`; `authUid`/`inviteStatus`/`firstSignInAt` are Cloud Function only |
| `playerLinks` | own doc only | never client-writable — Cloud Functions only (`verifySignInCode` / `linkAccount`) |
| `seasons` | any signed-in member | admin only |
| `sessions` | any signed-in member | admin only |
| `sessionEnrollment` | admin, or the player themselves (own doc, by `playerId` field) | never client-writable — the `setSeasonSignup` callable only, so every session doc for a player stays consistent |
| `weeks` | any signed-in member (staff: read-only) | admin only |
| `availability` | the player themselves + admin (not staff) | the player themselves (own doc only) + admin |
| `socialPlans` | any signed-in member, staff included | the player themselves or admin, only while the week isn't `complete`; `dinner`/`golfSim` must be none/before/after; delete admin only |
| `matchGroups` | admin always; everyone else only once the week is `matches_set` / `in_progress` / `complete` (keeps `pairing_draft` groups admin-only). Staff read-only | `sets` field only, by any of the group's `players`, only while `status` is `in_progress` or `reported` (never `locked`); everything else (incl. `setLineups`) admin / pairing Cloud Function only |
| `pairingRuns` | admin only | never client-writable — Cloud Function (Admin SDK) only |
| `changeRequests` | the requesting player + admin | create: any player, for themselves, while the week is `matches_set`; approve/deny: admin only |
| `eloHistory` | any signed-in player (not staff) — everyone sees everyone's history (TP-016) | never client-writable — Elo Cloud Function only |
| `joinRequests` | admin only | never client-writable — `submitJoinRequest` / `resolveJoinRequest` (Admin SDK) only |
| `signInCodes` | nobody | nobody — Cloud Functions (Admin SDK) only |

## Elo formulas (ported from the existing Excel system)
- K-factor: 32 (editable per season)
- Team Elo = average of the 2 players' individual Elos
- Expected score (Team A) = 1 / (1 + 10^((TeamB_Elo − TeamA_Elo) / 400))
- Margin multiplier = 0.6 + 0.16 × (|point_diff| − 1)
- Per-set adjustment = K × margin_multiplier × (actual_result − expected_score), same adjustment applied to both teammates. **Each set's teams come from that set's `setLineups` entry** (partners rotate every set), not from a fixed group pairing.
- Applied once per week at lock time (all 3 sets use start-of-week Elo as basis)
- Next-season regression: shrinkage = 0.7 × sets_played / (sets_played + 20); next_start = 1500 + shrinkage × (final_elo − 1500)

## Scoring & lock flow
- `matchGroups.status`: `scheduled` → `in_progress` → `reported` (editable) → `locked` (immutable)
- Either player writes `sets` while `in_progress`/`reported` — last write wins. (TP-011)
- **Lock Week** — designed, **not built yet**: an admin-only Cloud Function that would lock every `matchGroups.sets` for the week, compute Elo from each set's own `setLineups` teams, write `eloHistory`, and update `players.currentElo`. (TP-012)

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
- `weeklyAutomation` — a scheduled function checking every 15 minutes (America/Chicago) against each week's stored `automation` timestamps: builds the draft Wednesday 9 AM, sends Wednesday 5 PM unless held, finalizes Thursday 8 AM (TP-030). Automation times are computed once, at season setup, from each week's Thursday date.

## Season setup — implemented and deployed
`setupSeason` (admin callable, `functions/season/setupSeason.js`) creates a season, its sessions, and every Thursday week between each session's start/end date (skipping any given skip dates, e.g. Thanksgiving) in one batch. Idempotent — re-running it only touches weeks still in `draft`. Season 26-27 was created this way: Session 1 Oct 15 - Dec 17, Session 2 Jan 7 - Mar 11, Thanksgiving skipped, 19 weeks (TP-029).

## Roster, invites, and join requests — implemented and deployed
`functions/roster/roster.js`:
- `upsertPlayer` (admin) — create or edit a player; starting Elo can't change once they have a locked match.
- `sendInvites` (admin) — emails the install-first invite (see below) to selected players or every not-yet-invited player; flips `inviteStatus` to `invited` only for players not already signed in (TP-038) — a re-send to a signed-in player still emails them but never touches their status.
- `decline` (HTTP, rewritten from `/decline`) — a player's one-click "stop inviting me" link from the invite email; always returns the same branded page regardless of whether the token matched, so it never reveals anything to a guesser.
- `submitJoinRequest` (public, signed-out) — someone not on the roster asks to join; rate-limited per email per day; emails every admin.
- `resolveJoinRequest` (admin) — approve (creates the player + sends an invite) or deny.
- `setSeasonSignup` (the signed-in player, for themselves) — writes the player's `choice` to every session's `sessionEnrollment` doc for the active season in one batch; if opting out drops them from a session whose week is already `matches_set`/`in_progress` and they're in a match, raises a `swap_out` change request instead of silently dropping them (TP-027).

## Invite email — v2, "install-first" (TP-035)
`buildInviteEmail` in `functions/roster/roster.js` builds a table-based, inline-styled HTML email (Gmail/Outlook strip `<style>` and block SVG) plus a plain-text part that mirrors every section. Content, in order: a "not in the App Store" note, a Quick-start box (press-and-hold the button → Open in Safari), a picture of the Home Screen row (PNG, built from the real `apple-touch-icon`, served from `/email/` on hosting), 5 numbered iPhone install steps, separate Android steps, both sign-in methods, the current season's dates (pulled live from `sessions`, not hard-coded), and the existing DataForge footer. Sender display name is "Tom Junker" for this email only (every other email shows "TNPL") — `mailer.js`'s `sendMail` takes an optional `fromName` override for this.

## Email sending (TP-032)
`functions/email/mailer.js` sends through Tom's Gmail account (app password: `GMAIL_USER` / `GMAIL_APP_PASSWORD`); Resend support exists (`RESEND_API_KEY`) but is not enabled. All replies go to Tom's DataForge address; every email gets the same DataForge footer appended in one place.

## iPhone install gate (TP-036)
`src/lib/platform.js`: `isIOS()`, `isStandalone()` (display-mode / `navigator.standalone`), `iosBrowser()` (distinguishes Safari from Chrome/Firefox/Edge/Google-app/in-app browsers on iOS by user-agent token). `SignIn.jsx` renders a "First, add TNPL to your Home Screen" gate instead of the sign-in form whenever `isIOS() && !isStandalone()`, since Safari and the installed Home Screen app keep separate sign-in state. A visible "I'm on a computer" link bypasses the gate for the current browser session (`sessionStorage`, with a try/catch fallback). Desktop and Android are unaffected.

## Bottom navigation & Home (TP-037)
`src/components/BottomNav.jsx` — a fixed bottom tab bar (Home, Matches, Rankings, Players, Profile, plus Admin only when `isAdmin`), shown on every signed-in page. `src/pages/ComingSoon.jsx` is a placeholder for Matches/Rankings/Players until those pages are built, showing the active season's start date. `src/pages/Home.jsx` shows an "Are you playing {season}?" card until the signed-in player (not staff) has a season choice on record — gone for good once one exists, with a same-visit-only green confirmation right after choosing — plus a pre-season card (season name, start date, when week-1 availability opens) while today is before the first week.

## Roster status — `firstSignInAt` is the source of truth (TP-038)
`src/lib/rosterStatus.js`: `hasSignedIn(player)` is true if `firstSignInAt` is set OR `inviteStatus` is `accepted`; `classify(player)` derives the Admin Roster screen's Active/Invited/Inactive tab from that (Active = active && (signed in, or never invited); Invited = invite sent, no response yet; Inactive = `active: false` or declined). A one-time script, `functions/scripts/backfillSignIns.js` (dry-run by default, `--apply` to write, idempotent), sets `firstSignInAt`/`inviteStatus` for anyone with a `playerLinks` doc who's missing them — needed for accounts linked before this logic existed. **Known gap (ISS-014, priority 1):** "never invited" and "signed in" currently share the Active tab, which shows every never-invited player as if they were active; the tabs need a redesign to separate those two cases.

## Change requests
Player flags an issue with their assigned group (swap out / time change), or opting out mid-session raises one automatically (TP-027) → `pending` → admin approves/denies. **Not built yet:** the admin approve/deny UI (requests currently just queue up) and per-slot re-run on approval.

## Pages
Home · Matches · Rankings · Players · Profile · Admin (dashboard, season setup, roster, pairing draft/edit screens) · **SignIn** (route-guard redirect target; also renders the iPhone install gate — not a nav item). Bottom tab bar on every signed-in page (TP-037). Matches/Rankings/Players currently show a Coming Soon placeholder. Staff see the same tabs as a player, minus the season-signup card on Home.

## Local emulator testing
Firebase Local Emulator Suite (auth 9099, firestore 8080, functions 5001, UI) under the demo project ID `demo-tnpl`, so nothing can reach real Firebase resources (TP-026). Needs JDK 21+ (firebase-tools 15.x).
- Start: `firebase emulators:start --only auth,firestore,functions --project demo-tnpl`
- End-to-end check: `node functions/scripts/seedAndRunPairings.js` (repo root) — resets emulator state, seeds `functions/scripts/fixtures/roster.seed.json` (45 players, names + Season 3 Elos only) with synthetic `@example.test` emails, calls the callable as different users, runs 12 checks.
- Unit tests: `npm test` from the repo root (Node's built-in test runner, auto-discovers every `*.test.js`) runs both the 105 `functions/` tests and the client-side tests in `src/` (e.g. `src/lib/rosterStatus.test.js`) — 115 total. `npm test` inside `functions/` alone still works and runs just its own 105.
- One-time data scripts (`functions/scripts/`) — `importRoster.js`, `updatePhones.js`, `backfillSignIns.js` — all dry-run by default, `--apply` to write, real project or `--emulator`; see BestMethods.md.

## Known constraints / preferences
- Local dev on Windows 11, VS Code, PowerShell.
- Firebase config values (`VITE_FIREBASE_*`) go directly into `.env.local` by Tom — never relayed through chat.
