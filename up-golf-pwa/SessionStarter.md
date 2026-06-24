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
**Phase 7 COMPLETE ✅ — 2026 Outing COMPLETE ✅**
**Phase 8 is now the active phase — multi-tournament architecture + historical data.**

Tournament completed June 2026. All 4 rounds played. Tournament Lock pressed.
No code freeze in effect — Phase 8 development is active.

---

## What Was Completed This Session (2026-06-24)

- Historical data Excel file recovered (corrupt external links stripped) ✅
- UP_Golf_Historical_Import_v3.xlsx built — 6 sheets, all years 2011–2025 ✅
  - Tournaments, Players, Courses, Rounds, Results, Standings sheets
  - Ranks + payouts pre-filled from per-year sheets (2013–2025)
  - Course slope/rating pre-filled for all 5 courses from web research
  - gameFormat + handicapUsed pre-filled for all 46 rounds
  - Tees pre-filled for 2024/2025 R3+R4 from year sheets; older years yellow
  - Tom audited and confirmed file is final
- Tournament Results PDF export built (DEC-140) ✅
  - `TournamentResultsPDF.jsx` — new component at `/results-pdf` (admin) and `/history/:year` (players)
  - Admin Panel home screen: "📄 Export Results PDF" button (visible when status=complete)
  - Sections: header, age group winners, notable achievements, quick-nav links,
    final standings (all 32), top 5 overall/prize/skins, per-round leaderboards
    with match results, skins hole-by-hole (transposed: players as rows, holes as columns)
  - Print CSS hides bottom nav, sync bar, app chrome
  - Skins table shows opted-in players only (correct by design)
- History browser entry point added to Info screen (DEC-141) ✅
  - 🏆 History card added to Info screen grid
  - Navigates to `/history/2026` — player-accessible, no AdminGuard
  - Same TournamentResultsPDF component, no print button on player route
- GitHub URL fetch workflow implemented for all session docs ✅
  - All 7 UP Golf docs now in public dataforge-standards repo under up-golf-pwa/
  - MASTER_CLAUDE_PROTOCOL.md updated with new fetch workflow
  - CC prompts now always delivered as downloadable MD files (section 4 update)

---

## Next Session Priority

**Phase 8A — Multi-tournament architecture:**
1. Historical data import script (Node.js) — reads UP_Golf_Historical_Import_v3.xlsx,
   writes to Firestore `tournaments`, `historicalResults`, `historicalStandings`,
   `historicalPlayers` collections
2. History browser — `/history` route lists all tournaments by year; tapping navigates
   to `/history/:year` (already built)
3. `tournaments` collection + `config/active` Firestore doc
4. `useActiveTournament.js` hook replaces `ACTIVE_OUTING_ID` constant

**Results PDF — known remaining items:**
- Skins column cutoff (hole 18 + skins won) — landscape print fix deployed,
  verify in next test print

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

### Tournament Results PDF (DEC-140)
- Component: `src/components/TournamentResultsPDF.jsx`
- Routes: `/results-pdf` (AdminGuard) and `/history/:year` (player-accessible)
- Data sources: `useLeaderboard`, `useSkins`, `useOutingSetup`, round docs, score docs
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

---

## Session Workflow
1. Claude fetches all docs via URLs in Project Instructions at session start
2. Always upload `constants.js` and `firestorePaths.js` when working on hooks/utilities
3. **NEVER read keys/tokens/IDs from screenshots — always require copy/paste**
4. **Always wait for all necessary files/info before writing code**
5. Work through issues/features
6. At session end: say "Update the docs" → Claude generates CC prompt MD files
