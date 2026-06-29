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
**Phase 8A** is underway. Historical data (2011–2025) imported to Firestore history_ collections. History browser (Tournaments + Players tabs) live at /history. Off-season 3-tab nav (Home | History | Info) implemented. App renamed to Tourney Golf. Archive Tournament admin function built.

---

## Last Session — June 29, 2026

### Completed
- **Historical data import** — `UP_Golf_Historical_Import_v3.xlsx` imported via Node.js Admin SDK scripts (`fetch-uids.js` + `import-history.js`) into 4 Firestore collections:
  - `history_tournaments` (14 docs, 2011–2025)
  - `history_rounds` (53 docs)
  - `history_results` (1,151 docs)
  - `history_standings` (303 docs)
  - Active players linked by UID; historical-only players stored by name only
- **Off-season nav** — `useOutingStatus.js` hook; 3-tab nav (Home | History | Info + Admin) when `outing.status === 'complete'`; 5-tab nav during active tournament
- **History screen** (`/history`) — Tournaments tab (year picker → tournament detail with rounds table + standings table + per-round scores) + Players tab (alphabetical list + search + career stats: years played, total earned, best finish, top 3, course history, year-by-year)
- **Tabler Icons** — added via CDN in index.html; `ti ti-history` used for nav and header icons
- **App renamed** — "UP Golf" → "Tourney Golf" in index.html title and apple-mobile-web-app-title
- **Firestore rules** — added read (isSignedIn) + write (isAdmin) for all 4 history_ collections
- **Composite indexes** — created for history_standings (year+finalRank) and history_rounds (year+roundNumber)
- **Course name fix** — "Grey Walls" corrected to "Greywalls" in 5 Firestore docs (2012–2016 R2)
- **2011 standings** — added history_standings docs for Tom Junker, Brad Pietz, Mike Dethloff with finalRank: null
- **Archive Tournament** — `AdminArchiveTournament.jsx` admin component; reads live outing data, writes to history_ collections; only visible when outing.status === 'complete'
- **"Full Results" button** — in History tournament detail view; navigates to /results-pdf; only shows for current active outing (matched by outingId)

### Deferred to Phase 8B
- `config/active` Firestore doc + dynamic outingId (currently hardcoded as ACTIVE_OUTING_ID)
- `TournamentResultsPDF` reading outingId from URL param (currently always uses ACTIVE_OUTING_ID)
- History doc ID migration: year-based ("2025") → outingId-based ("outing_2025")
- Tournament creation wizard
- Off-season Home screen redesign
- "Create New Tournament" admin form
- Multiple tournaments per year support

### Known Issues / Notes
- BestMethods.md raw URL must use `refs/heads/main` not `main`:
  `https://raw.githubusercontent.com/Whit19/dataforge-standards/refs/heads/main/up-golf-pwa/BestMethods.md`
- history_tournaments docs for 2011–2025 use year as doc ID ("2025"); future tournaments will use outingId ("outing_2026") — migration needed in Phase 8B
- TournamentResultsPDF always loads ACTIVE_OUTING_ID regardless of URL — Phase 8B fix
- 2026 outing not yet archived to history (Archive button built, ready to run when confirmed)

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
