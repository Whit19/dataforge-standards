# SessionStarter.md — UP Golf PWA
**Protocol:** Load MASTER_CLAUDE_PROTOCOL.md before this file.
Repo: github.com/Whit19/dataforge-standards
**Load this at the start of every session to restore full context.**

---

## What We're Building
UP Golf — a Progressive Web App for an annual multi-day golf outing. 32 players, 
3 admins (Tom Junker, Mike Dethloff, Brad Pietz), 4 rounds across 3 days. Features: 
scoring, leaderboards, skins, team matches, prize pools, chat, photo sharing, push 
notifications.

**Firebase project ID:** `up-golf-pwa`
**Local project path:** `C:\Dev_Projects\up-golf`
**Stack:** React + Vite, Firebase (Firestore, Auth, Storage, Cloud Functions, Hosting), 
GoDaddy CNAME subdomain.

---

## Current Status
**Phase 8A** is underway. outing_2026 successfully archived to history_ collections with correct purse/location/format/winnings data (was broken — see last session). functions/prizes.js rewritten to persist match/scramble/skins winnings to Firestore instead of only computing them live; AdminPayments.jsx now reads those persisted values. TournamentResultsPDF.jsx now supports viewing archived years via /history/:year (tested against 2025), with a new full payout-breakdown-by-category table. Historical data (2011–2025) imported to Firestore history_ collections. History browser (Tournaments + Players tabs) live at /history. Off-season 3-tab nav (Home | History | Info) implemented. App renamed to Tourney Golf.

---

## Last Session — June 30, 2026

### Completed
- **outing_2026 archive fixed and re-run** — `AdminArchiveTournament.jsx` was
  reading three fields that don't exist on live docs (`payments.prizeWon`/
  `skinsWon`, `outing.location`/`startDate`/`endDate`, `rounds.gameFormat`).
  All fixed (DEC-155, ISS-141). Archive re-run successfully — History screen
  now shows correct format/course/dates/standings/$4,320 total purse for 2026.
- **functions/prizes.js rewritten** — the Cloud Function was running
  successfully on every round lock (confirmed via Cloud Run logs) but two of
  its four internal write steps were silently producing zero results despite
  logging success and permanently setting the idempotency guard
  (`prizesCalculatedAt`). Root caused and fixed:
  - Match results (R2/R4): batch updates now counted; throws if matches exist
    but none could be scored, instead of committing an empty batch (DEC-156)
  - Scramble results (R3): place-assignment tie-detection only compared
    `totalGross`, ignoring the `backNine`/`lastThree` tiebreakers the sort
    above it already used — collapsed 3 distinctly-ranked tied teams into one
    shared place, paying the wrong prize tier. Fixed to use the full
    tiebreaker chain (DEC-157)
  - `prizePerPlayer` now written directly onto `matches` docs; hardcoded
    `/8` match count replaced with actual per-round count (DEC-156)
  - `prizesCalculatedAt` idempotency write now gated on real results existing,
    so a future failure can retry instead of being permanently skipped
- **AdminPayments.jsx switched to reading persisted data** — `CollectPaySection`
  no longer recomputes match winners/prizes/scramble winnings live client-side;
  reads `matches.winner`/`prizePerPlayer` and `payments.scrambleWinnings`
  directly (DEC-158). This pattern (live recompute as a fallback for a
  disabled Cloud Function, never reverted after the function was fixed) had
  been silently diverging from Firestore for ~3 months with no visible symptom
  — only discovered because the archive script read the wrong data while the
  UI kept showing the right numbers.
- **Verification process** — one-time snapshot script captured live UI
  winnings as ground truth before any code changed; backfill script manually
  invoked the new calculation logic against the already-locked outing_2026
  and diffed against the snapshot. Final result: 32/32 players matched exactly.
  Both one-time scripts deleted after use per convention.
- **TournamentResultsPDF.jsx: historical mode** — new `useHistoricalResults.js`
  hook reads `history_` collections and reshapes them to match
  `useLeaderboard`'s output shape. Component now branches on whether the
  `:year` URL param matches the currently active outing; if not, reads
  archived data instead of always defaulting to `ACTIVE_OUTING_ID`. Cover
  header/print bar/footer year display also fixed to match (was hardcoded to
  `ACTIVE_OUTING_YEAR` even in historical mode). Tested against `/history/2025`.
- **Payout Breakdown table added** — new section in TournamentResultsPDF
  showing every player's winnings broken out by category (R1/R2/R4 skins, R2/R4
  match, R3 scramble in live mode; R1–R4 totals in historical mode, since
  `history_results` doesn't store category-level detail within a round) (DEC-160).

### Deferred to Phase 8B
- `config/active` Firestore doc + dynamic outingId (currently hardcoded as ACTIVE_OUTING_ID)
- History doc ID migration: year-based ("2025") → outingId-based ("outing_2025")
- Tournament creation wizard
- Off-season Home screen redesign
- "Create New Tournament" admin form
- Multiple tournaments per year support
- TournamentResultsPDF historical mode: per-round leaderboard detail tables
  and hole-by-hole skins grids not yet built for archived years (ISS-148)
- ageGroup not present in history_standings schema — Grp column shows '—'
  for archived years until added to the import/archive schema

### Known Issues / Notes
- BestMethods.md raw URL must use `refs/heads/main` not `main`:
  `https://raw.githubusercontent.com/Whit19/dataforge-standards/refs/heads/main/up-golf-pwa/BestMethods.md`
- history_tournaments docs for 2011–2025 use year as doc ID ("2025"); future
  tournaments will use outingId ("outing_2026") — migration needed in Phase 8B
- The original cause of why functions/prizes.js's first invocation silently
  wrote zero match results (before the loud-failure fix) was never fully
  root-caused — the fix prevents recurrence but the historical mechanism
  remains unknown (ISS-149, logged for awareness only)
- DEC-136 flagged a citation error: SessionStarter previously cited "DEC-122"
  for a code-freeze decision that doesn't match DEC-122's actual content
  (r3Leaderboard derivation). No dedicated DecisionLog entry for the freeze
  itself was ever found. This reference has now been removed from this file
  since the freeze it described is no longer current/relevant context.

---

## Phase 8 Summary (Post-UP-Outing — fully scoped, DEC-101–105)

- **8A:** Multi-tournament — `config/active` Firestore doc + `useActiveTournament.js` 
  hook replaces `ACTIVE_OUTING_ID`; `tournaments` + `enrollments` collections;
  historical data import; history browser
- **8B:** Course data — GolfCourseAPI.com (free) or GolfAPI.io (paid); fetched once 
  at admin setup, stored in Firestore `courses` collection
- **8C:** Net scoring — `handicapIndex` on `users` doc (admin-entered); WHS Course 
  Handicap formula; `useNetScoring.js` hook
- **8D:** Flexible formats — data-driven `roundFormats[]` on tournament doc
- **8E:** Admin UX — handicap entry, course handicap display, stroke indicators

---

## Key Architecture Notes

### Historical Data Collections (Phase 8A — new)
Import script writes to these Firestore collections:
- `historicalTournaments/{year}` — tournament metadata (name, dates, courses, purse)
- `historicalResults/{year}_{roundNumber}_{playerName}` — per-round scores, tees, payout
- `historicalStandings/{year}_{playerName}` — final rank + total payout per year
- `historicalPlayers/{playerName}` — career data (years played list, yearsCount)
- `courses/{courseName}_{tees}` — slope/rating per tee per course (already in import file)
Source: `UP_Golf_Historical_Import_v3.xlsx` (6-sheet Excel, Tom has audited/finalized)

### Tournament Results PDF (DEC-140, DEC-159, DEC-160)
- Component: `src/components/TournamentResultsPDF.jsx`
- Routes: `/results-pdf` (AdminGuard, always live data) and `/history/:year`
  (player-accessible — live data if :year matches the active outing's year,
  otherwise reads archived history_ collections via `useHistoricalResults.js`)
- Live mode data sources: `useLeaderboard`, `useSkins`, `useOutingSetup`,
  round docs, score docs, payments docs
- Historical mode data source: `useHistoricalResults.js` (reads
  history_tournaments/history_rounds/history_results/history_standings)
- Includes a Payout Breakdown table (per-player, per-category in live mode;
  per-player, per-round in historical mode)
- Skins table: transposed (players as rows, holes 1–18 as columns, skins won count last)
- Skins participants: opted-in players only (from payments collection skinsOptIns)
- Print CSS: hides `.bottom-nav`, `.sync-bar`, `.app__main` padding
- Skins sections use `@page landscape` for width; rest portrait
- `useLocation` detects route — print button only on `/results-pdf`

### History Browser (DEC-141)
- Info screen: 🏆 History card navigates to `/history/2026`
- When Phase 8A builds `historicalTournaments` collection, `/history` route will
  render a year-picker list; `/history/:year` renders that year's results
- `/history/:year` currently renders TournamentResultsPDF for 2026 live data;
  will be extended to support historical data once import is complete

### Firestore Doc ID Conventions (CRITICAL)
All doc IDs must match exactly between `firestorePaths.js` and `generatePairings.js`:
- Foursomes: `{outingId}_R{roundNumber}_G{groupNumber}`
- Matches: `{outingId}_R{roundNumber}_M{matchNumber}`
- Scores: `{outingId}_R{roundNumber}_{playerId}`
- Scramble teams: `{outingId}_T{teamNumber}`
- Rounds: `{outingId}_R{roundNumber}`

### Push Notification Architecture (fully validated ✅)
- FCM tokens in `users/{uid}.fcmTokens` (array, replaced not appended)
- All notifications handled by SW native `push` event — no Firebase compat SDK in SW
- Token refresh: `checkAndRefreshToken` runs silently on every mount

### Player Credentials
- Username: full first + last name, mixed case (e.g. `TimFischer`)
- Password: firstname + `26!`, lowercase (e.g. `tim26!`) — admins use `Sweetgrass`
- App auto-appends `@upgolf.local` — players never type the email suffix

### Auth Pattern
```js
const { user, isAdmin } = useAuth()
import { useAuth } from '../hooks/useAuth.jsx'  // .jsx required
```

### PowerShell
No `&&` chaining. Always `npm install --legacy-peer-deps`. Always `npm run build` 
before `firebase deploy --only hosting`. Functions require separate 
`firebase deploy --only functions`.

### VAPID Key (copy/paste only — never retype)
BLcR9lmWMj4kPIpMoo-jT88mu9ZoBO6dIGeGcjmhiNXhLhc0f8Oat8HaLGaxc1x03_p8Ji8EhJFb_arYA-RyN-A

---

## Phase Overview

| Phase | Status |
|-------|--------|
| Phase 0–7 | ✅ Complete |
| Phase 8 | 🟡 In Progress |

---

## Open Issues Summary
- **ISS-010** Emoji picker (deferred — post Phase 8)
- **ISS-148** TournamentResultsPDF historical mode missing per-round/skins detail (deferred — Phase 8B)
- **ISS-149** Original cause of prizes.js silent empty-batch writes never fully root-caused (logged for awareness, not blocking)

---

## Session Workflow
1. Claude fetches all docs via URLs in Project Instructions at session start
2. Always upload `constants.js` and `firestorePaths.js` when working on hooks/utilities
3. **NEVER read keys/tokens/IDs from screenshots — always require copy/paste**
4. **Always wait for all necessary files/info before writing code**
5. Work through issues/features
6. At session end: say "Update the docs" → Claude generates CC prompt MD files
