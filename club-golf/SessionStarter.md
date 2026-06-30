# SessionStarter.md — Club Golf Platform
**Protocol:** Load MASTER_CLAUDE_PROTOCOL.md before this file.
Repo: github.com/Whit19/dataforge-standards
**Load this at the start of every session to restore full context.**
*Updated June 2026 — Session 32 complete (Review step team display bug fixed — CGI-093)*

---

## What We're Building
Club Golf — a player-first, multi-tenant PWA for golf game scoring, format variety, and personal history. Initial focus: 6-Point Game at Tripoli Golf Club. Platform goal: multi-club, format library, EZ Games, subscription monetization.

**The north star: "The player's app — not the pro's app."**

**Firebase project ID:** `club-golf-9f946`
**Local project path:** `C:\Dev_Projects\club-golf`
**Stack:** React 18 + Vite, Firebase (Firestore, Auth, Hosting, Cloud Functions), same patterns as UP Golf PWA
**Club ID (v1):** `club-golf-v1` (stored in localStorage key: `cg_club_id`)

---

## Background
Built by Tom Junker, who also built UP Golf PWA (fully deployed, 32 players, Phase 7 in progress). All core patterns proven. Do not re-litigate proven architecture decisions.

**Platform vision defined May 2026:**
- Club Golf is the platform; "Club Golf" is the brand (no club-name suffix in v1)
- Each club gets its own Firestore-scoped instance
- Player-first: any member can create games; no pro shop control layer
- Format library with configurable options (not fully custom engines)
- One primary format + optional side games per game
- Scheduled → Active → Complete game lifecycle
- Hybrid monetization: free tier + Pro subscription + per-event option

---

## Current Status
**Session 32 complete. ~72h invested.**

### What's working (all prior sessions + Session 31):
- Full 6-Point Game scoring flow
- Read-only viewer link
- ResultsPage + HistoryPage + statsUtils
- BottomTabBar — 5 tabs for all users (Home / History / Leaderboard / Profile / Info); tab bar hidden on home page
- AdminPage — Roster + Course + Club Profile management
- Member invite flow — sendInvite / claimInvite / deleteMember / deactivateMember / reactivateMember Cloud Functions
- PWA icon + manifest + "Add to Home Screen" prompt
- Game creation open to all authenticated members
- Landscape scorecard view, Firestore security rules — roles-based
- ProfilePage, HistoryPage, GamesPage (single scrolling feed)
- Multi-course support, scheduled games, plus handicaps
- LeaderboardPage — dedicated tab with auto-detected year picker, 4 stat tabs
- Per-year stats — `stats.all` (lifetime) + `stats[year]`
- Deactivate / Reactivate member, role editor in RosterPage
- HI Age indicator — color-coded badge in NewGamePage and RosterPage
- Custom domain `club-golf.app`, PWA update banner, invite email overhaul
- Game creator access control — only creator or admin can edit; others redirected to viewer
- GamesPage redesigned as home page — windmill header, stats strip, New Game hero button, 2×2 nav grid
- Live game card hole/score display via `getLiveGameStatus()` helper
- Viewer back navigation — portrait + landscape
- GHIN admin sync — home page modal, full roster, weekly reminder email
- **NewGamePage redesigned as 4-step wizard: Setup / Teams / Settings / Review**
  - Setup step: course dropdown (always shown), date, compact player rows (name + inline HI edit + age badge + tee dropdown), GHIN sync button when all 4 filled
  - Tees default to logged-in member's `defaultTees`; guests also default to creator's tees
  - Inline HI editing — tap HI value → input in place → blur commits → writes to member doc immediately (guests: local only)
  - Guest name entry — inline editable text input in the name column
  - Teams step: player pool with PH, 2×2 slot grid, combined PH balance indicator, team colors (A=blue, B=orange)
  - Team assignment uses explicit slot picker — tap slot → mini picker shows unassigned players
  - Teams reset when player slots change
  - Settings + Review steps: unchanged
  - GHIN external lookup link removed
  - Review step team display now reads live `teamAssignment` state via `teamForSlot(s.slotId)` instead of stale `s.team` slot property (CGI-093, fixed 6/30/26)
- **Per-game GHIN sync in Setup step**
  - "Update handicaps from GHIN" button — appears only when all 4 players filled
  - Scoped to the 4 players in the game (memberIds filter + guestNames)
  - Cascading search for roster members: Tripoli+WI → WI only
  - Guest search: single state (creator selects in modal, defaults to All States)
  - Guest state selector in GHIN modal — per guest, full US state list alphabetical, "All States" first
  - GHIN modal shows roster member names (auto lookup) vs guest state selectors
  - `parseGhinHI` helper — treats "NH" and non-numeric HI values as null
  - Single result with null HI → cascades to next level rather than stopping
  - Multiple results with same non-null HI → auto-resolved
  - Multiple results with different HIs → disambiguation picker (name + club + state)
  - Ambiguous guests → `ambiguousEntries` (not `updates`) — no NaN display
  - Guest lookup errors → surfaced as empty candidate entry with "could not look up" message
  - Result display: named updated list (green) + not found list (amber) + disambiguation picker
  - Roster member writes: `handicapIndex` + `handicapUpdatedAt` to member doc
  - Guest writes: local slot state only, not saved to Firestore
  - Full-roster summary (`lastHiSync`) only written for admin sync, not per-game sync

### Known remaining items:
- [ ] Jon Cyganiak — GHIN name is "Jon I. Cyganiak"; CF splits to last="I. Cyganiak" → not found. Update HI manually in RosterPage or rename member doc to "Jon I. Cyganiak"
- [ ] Begin inviting members — work through To Invite tab in RosterPage
- [ ] Monitor invite delivery from tjunker9@gmail.com
- [ ] crack.mp3 audio file (source and place at `public/sounds/crack.mp3`)
- [ ] Delete serviceAccountKey.json from scripts/ after confirming no longer needed
- [ ] Send wave 1 invite emails once members have installed PWA

### Next session priorities:
1. Begin sending member invites through RosterPage To Invite tab
2. Monitor delivery / Safari link issues
3. Verify Firestore-saved team assignments on a couple of real games created before the CGI-093 fix (display bug only — save logic was unaffected, but worth a spot check)
4. Phase 3 — Thursday Night Men's League

---

## Platform Roadmap Summary

| Phase | Name | Status | Est. Hours |
|-------|------|--------|-----------|
| 1G | Admin + Deploy | ✅ Complete | — |
| 1H | Club Profile & Platform Foundation | ✅ Complete | — |
| 2A–2I | Full Member Roster + Auth + GHIN Sync | ✅ Complete | — |
| 3 | Thursday Night Men's League | 🔲 Not Started | 12–18h |
| 4 | Format Library + EZ Games | 🔲 Not Started | 30–45h |
| 5 | Platform Launch + Monetization | 🔲 Not Started | 20–30h |

---

## Data Model Notes (critical — don't forget)
- `holes[n].scores[playerId]` is `{ gross, net }` object — NOT a plain number
- Member doc IDs are auto-generated (NOT Firebase Auth UIDs) — uid is stored as a field
- `member.stats` is NESTED — `stats.all` (lifetime) + `stats[year]` (e.g. `stats.2026`)
- `member.handicapIndex` — number | null — blank stays null, never 0
- `member.handicapUpdatedAt` — Firestore Timestamp | null
- `member.hiSyncStatus` — 'synced' | 'not_found' | 'error' | null — written by syncHandicaps
- `clubs/{clubId}/profile/profile` — correct 4-segment path for club profile doc
- `lastHiSync` field on club profile doc: `{ ranAt, updatedCount, notFoundCount, errorCount, ranBy, dryRun }`
- Plus handicaps stored as negative numbers (e.g. -3.5 for +3.5 HI)
- `GAME_STATUS` three values: `SCHEDULED`, `ACTIVE`, `COMPLETE`
- Team assignment stored per player via `teamForSlot(slotId)` — reads from `teamAssignment` state, not slot.team

---

## Key Product Decisions (Final)

| ID | Decision |
|----|----------|
| CGAD-001 | Multi-tenant from day one — all data under clubs/{clubId} |
| CGAD-049 | Game edit access restricted to creator + admins |
| CGAD-050 | Home page has no bottom tab bar; Games tab renamed to Home |
| CGAD-051 | GHIN sync is admin-triggered (not scheduled); weekly reminder email Monday 8 AM Central |
| CGAD-052 | NewGamePage redesigned as 4-step wizard: Setup / Teams / Settings / Review |
| CGAD-053 | Inline HI editing with immediate Firestore write-back; guests local only |
| CGAD-054 | Per-game GHIN sync scoped to 4 players; cascading search; guest state picker; disambiguation |

Full rationale in DecisionLog.md. Next available ID: **CGAD-055**

---

## Tech Stack Notes
- **Production URL:** `https://club-golf.app`
- React 18 + Vite
- Firebase (Firestore, Auth, Hosting, Cloud Functions — Node.js 24, v2 API)
- `npm install --legacy-peer-deps` always
- PowerShell: no `&&` chaining
- All Firestore refs through `firestorePaths.js`
- Cloud Functions: `syncHandicaps` timeout 180s; `sendHiSyncReminder` scheduled every monday 13:00 UTC
- Club profile doc path: `clubs/{clubId}/profile/profile` (4 segments)
- GHIN API: unofficial consumer API — credentials passed at call time, never stored
- GHIN `parseGhinHI` helper: treats "NH" and non-numeric values as null
- GHIN roster search cascade: Tripoli+WI → WI only
- GHIN guest search: single state passed from client (guest.state), defaults to null (all states attempted)
- GHIN `syncHandicaps` return shape: `{ updated, notFound, errors, notFoundNames, errorNames, dryRun, updates, guestUpdates, ambiguous }`

---

## Session Workflow
1. Load this SessionStarter + upload all 6 docs
2. Also load: BestMethods.md — read relevant section before starting any new feature
3. Design/data model decisions in this session first
4. Code delivered as CC prompts for VS Code Claude
5. Upload files when debugging logic bugs
6. At session end: update all docs
