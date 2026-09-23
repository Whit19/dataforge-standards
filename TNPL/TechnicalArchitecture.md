# TNPL — Technical Architecture

**Last updated:** 2026-09-23

## Stack
- Frontend: React + Vite
- Backend / data: Firebase — Firestore (data), Firebase Auth (Google + Email Link providers), Firebase Hosting (deploy), Cloud Functions (pairing engine — built; Elo calc, week-lock — not yet)
- Firebase project: `tnpl-pwa` (console: https://console.firebase.google.com/u/1/project/tnpl-pwa/overview)
- Repo: https://github.com/Whit19/TNPL (private)
- Local folder: `C:\Dev_Projects\TNPL`
- Docs: `C:\Dev_Projects\dataforge-standards\TNPL\` (cloned locally)
- CLI config: `firebase.json` (firestore rules/indexes, `functions` source, emulators block), `.firebaserc`, `firestore.indexes.json`
- **Nothing is deployed yet** — rules, functions, and hosting are all local/emulator only.

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
  currentElo                            // player role only; missing = pairing defaults to 1500
  authUid                               // set on first sign-in once matched

playerLinks/{authUid}
  playerId
  // Reverse index, auth UID -> playerId. Firestore security rules can only
  // do get() by a known document path, not arbitrary queries — this is what
  // lets isAdmin()/isPlayerRole() rules helpers resolve "which player is the
  // caller" from request.auth.uid alone. Written by auth.js alongside the
  // authUid field on the matched player doc. (TP-014)

seasonEnrollment/{seasonId}_{playerId}
  optedIn: boolean, startingElo, enrolledAt

seasons/{seasonId}
  startDate, endDate, kFactor
  pairingFlagThreshold                  // default 100 (TP-018)
  slotFairnessTolerance                 // default 25 (TP-021)

weeks/{weekId}
  seasonId, date
  status: draft | availability_open | pairing_draft | matches_set | in_progress | complete
  timeSlots: [{ id, label: "6:00 PM", courtsAvailable: 2 }, ...]   // slot order = array order

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

eloHistory/{weekId}_{playerId}
  eloBefore, eloAfter, delta
```

No locked-partner/couples constraint (TP-008).

## Auth & roles — implemented
- Firebase Auth, Google + Email Link (passwordless) providers. (TP-010)
- `src/auth.js`: `signInWithGoogle()`, `sendSignInLinkToEmail()` / `completeEmailLinkSignIn()` (handles the cross-device case — link opened without a stored email locally surfaces a `needs_email` state), `lookupPlayerByEmail()`.
- On successful auth: match found in `players` by email → `authUid` written to that player doc + `playerLinks/{authUid}` reverse-index doc created. No match → user is signed back out, `not_on_roster` state returned (invite-only via roster, no dangling half-authenticated session).
- `role: 'staff'` — club workers with schedule visibility only, no Elo/pairing/availability/season enrollment.
- `isAdmin: true` — single admin (Tom) in v1.

## Firestore rules — implemented, not deployed
`firestore.rules` covers every collection below. Helpers `isSignedInPlayer()` / `isAdmin()` / `isPlayerRole()` centralize the `playerLinks` lookup rather than repeating it per rule. Validated via `firebase deploy --only firestore:rules --dry-run`.

| Collection | Read | Write |
|---|---|---|
| `players` | any signed-in member | admin only, except a player may edit their own `phone` (TP-016) |
| `playerLinks` | own doc only | written only by the app's auth flow (not general client writes) |
| `seasonEnrollment` | the player themselves + admin | the player themselves (own doc only, so they can opt in/out) + admin (TP-016) |
| `seasons` | any signed-in member | admin only |
| `weeks` | any signed-in member (staff: read-only) | admin only |
| `availability` | the player themselves + admin (not staff) | the player themselves (own doc only) + admin |
| `socialPlans` | any signed-in member, staff included | the player themselves or admin, only while the week isn't `complete`; `dinner`/`golfSim` must be none/before/after; delete admin only |
| `matchGroups` | admin always; everyone else only once the week is `matches_set` / `in_progress` / `complete` (keeps `pairing_draft` groups admin-only). Staff read-only | `sets` field only, by any of the group's `players`, only while `status` is `in_progress` or `reported` (never `locked`); everything else (incl. `setLineups`) admin / pairing Cloud Function only |
| `pairingRuns` | admin only | never client-writable — Cloud Function (Admin SDK) only |
| `changeRequests` | the requesting player + admin | create: any player, for themselves; approve/deny: admin only |
| `eloHistory` | any signed-in player (not staff) — everyone sees everyone's history (TP-016) | never client-writable — Elo Cloud Function only |

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
- **Lock Week** — admin-only Cloud Function: locks every `matchGroups.sets` for the week, computes Elo from each set's own `setLineups` teams, writes `eloHistory`, updates `players.currentElo`. (TP-012)

## Pairing engine — implemented (Cloud Function, undeployed)
Pure module `functions/pairing/engine.js` (CommonJS, no Firestore access, deterministic — all tie-breaks by playerId / slot order) plus the admin-only callable `generatePairings({ weekId })` in `functions/pairing/callable.js`, wired in `functions/index.js` (region us-central1, the default). Design: TP-017 through TP-023.

Engine steps:
1. **Overflow** — if more available players than capacity (courts × 4, normally 24), keep the earliest responders; the rest are `unplaced: overflow`.
2. **Fill** — if the count isn't a multiple of 4, add second-match entries from `canPlayTwo` volunteers (best combination by resulting spread); if not enough volunteers, bench the latest responders (`no_volunteer_fill`).
3. **Phase 1** — tightest Elo groups (sorted-Elo chunks of 4), repaired by swaps for hard constraints only: no duplicate player in a group; groups fit slots (no blocked slot, per-slot court capacity, a 2-match player's two groups in different slots). A player who can't be placed anywhere is `no_feasible_slot`.
4. **Slot assignment** — enumerates every valid group→slot arrangement; cost per player = times already played that slot this season (from prior published weeks' `matchGroups`), a preferred slot costs `PREFERRED_SLOT_BONUS = -1`; plus repeat-grouping penalties (same-week repeat 1000, prior-season pairing 0.5 each).
5. **Phase 2** — swap entries between groups to lower total fairness cost, accepted only if hard constraints hold, max and total spread stay within `slotFairnessTolerance` of Phase 1, and no group exceeds `max(phase1MaxSpread, pairingFlagThreshold)`.
6. **Output** — groups (players Elo-desc, `setLineups` from `SET_ROTATION`, `eloSpread`, `flagged` = spread > threshold), unplaced, twoMatchPlayers, slotTallies, stats.

Callable: requires an admin (resolved via `playerLinks` → `players.isAdmin`); week must exist and be `availability_open` or `pairing_draft`; loads availability (`canPlay`), enrollment (`optedIn`), players (missing `currentElo` → 1500, listed in `defaultEloUsed`), and slot/partner history from earlier `matches_set` / `in_progress` / `complete` weeks of the same season; then one batch replaces any prior draft groups, writes the new groups + `pairingRuns/{weekId}`, and sets the week to `pairing_draft`. A missing `respondedAt` counts as the latest responder. Not built yet: publish action (`pairing_draft` → `matches_set`), per-slot re-run on change-request approval, notifications, admin UI.

Constants: `functions/paths.js` is the single source of collection names and status strings inside `functions/`; a unit test asserts it (and `SET_ROTATION`) matches `src/constants.js`. After the first functions deploy, public invoker access must be set manually in the Cloud Run console.

## Change requests
Player flags an issue with their assigned group (swap out / time change) → `pending` → admin approves/denies → approval re-triggers pairing for just that slot (per-slot re-run not built yet).

## Pages
Home · Players · Rankings · Matches · Admin (change-request queue, Lock Week; planned: pairing draft/publish screen showing groups, flags, overflow/unplaced, per-player slot tallies, unmet preferred slots) · **SignIn** (route-guard redirect target, added during auth implementation — not a nav item). Staff see Home/Matches only, read-only. A weekly dinner / golf-sim section (admin + staff + players) is planned.

## Local emulator testing
Firebase Local Emulator Suite (auth 9099, firestore 8080, functions 5001, UI) under the demo project ID `demo-tnpl`, so nothing can reach real Firebase resources (TP-026). Needs JDK 21+ (firebase-tools 15.x).
- Start: `firebase emulators:start --only auth,firestore,functions --project demo-tnpl`
- End-to-end check: `node functions/scripts/seedAndRunPairings.js` (repo root) — resets emulator state, seeds `functions/scripts/fixtures/roster.seed.json` (45 players, names + Season 3 Elos only) with synthetic `@example.test` emails, calls the callable as different users, runs 12 checks.
- Unit tests: `npm test` inside `functions/` (20 tests, Node built-in runner).

## Known constraints / preferences
- Local dev on Windows 11, VS Code, PowerShell.
- Firebase config values (`VITE_FIREBASE_*`) go directly into `.env.local` by Tom — never relayed through chat.
