# SessionStarter.md — Club Golf Platform
**Protocol:** Load MASTER_CLAUDE_PROTOCOL.md before this file.
Repo: github.com/Whit19/dataforge-standards
**Load this at the start of every session to restore full context.**
*Updated June 2026 — Session 30 complete (GHIN handicap sync)*

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
**Session 30 complete. ~67h invested.**

### What's working (all prior sessions + Sessions 29–30):
- Full 6-Point Game scoring flow
- Read-only viewer link
- ResultsPage + HistoryPage + statsUtils
- BottomTabBar — 5 tabs for all users (Home / History / Leaderboard / Profile / Info); Games tab renamed to Home with ti-home icon; tab bar hidden on home page (/games exact)
- Admin tab removed from bottom bar — Admin Panel button in ProfilePage (admin only)
- AdminPage — Roster + Course + Club Profile management
- Member invite flow — sendInvite / claimInvite / deleteMember / deactivateMember / reactivateMember Cloud Functions
- Invite emails sent from tjunker9@gmail.com
- PWA icon + manifest + "Add to Home Screen" prompt
- Game creation open to all authenticated members
- Landscape scorecard view
- Firestore security rules — roles-based
- ProfilePage, HistoryPage, GamesPage (single scrolling feed)
- Multi-course support, scheduled games, plus handicaps
- Invite flow hardened, forgot password, password reset from RosterPage
- CSV bulk import — 183 members imported
- Lighthouse audit — Accessibility 94, Best Practices 100
- Starting hole + numHoles (9/18) settings with wraparound hole sequence
- LeaderboardPage — dedicated tab with auto-detected year picker, 4 stat tabs, full ranked list
- Per-year stats — member stats now nested as `stats.all` (lifetime) + `stats[year]`
- Deactivate / Reactivate member — admin can disable/re-enable Firebase Auth accounts from RosterPage
- Role editor in RosterPage — admin can change member role (member/admin) from edit panel
- RosterPage tab order — Active | Pending | To Invite | Not Active
- HI Age indicator — color-coded badge in NewGamePage and RosterPage
- Custom domain — `club-golf.app` live via Firebase Hosting + GoDaddy DNS
- PWA update banner — useRegisterSW hook in App.jsx
- Invite email overhaul — URL updated to club-golf.app, install instructions added
- Game creator access control — only creator or admin can edit; others redirected to viewer
- GamesPage redesigned as home page — windmill header, stats strip, New Game hero button, Games card, 2×2 nav grid
- **Live game card hole/score display** — home page live game rows show "Hole {n} · {A}-{B} · {date}" via `getLiveGameStatus()` helper in GamesPage.jsx; no new Firestore reads
- **Viewer back navigation** — ViewerPage has a back button in portrait (rp-back-btn) and landscape (sc-back-btn via optional onBack prop on ScorecardView) navigating to /games; ScoreEntryPage unaffected
- **GHIN handicap sync** — admin-triggered sync from home page; syncHandicaps Cloud Function fetches HI for all non-deactivated members from GHIN API; writes handicapIndex + handicapUpdatedAt + hiSyncStatus to each member doc; writes lastHiSync summary to club profile doc; sendHiSyncReminder scheduled function emails tjunker9@gmail.com every Monday at 8 AM Central; NOT FOUND / ERROR badges in RosterPage HI column; dry run mode supported

### Known remaining items:
- [ ] Jon Cyganiak — GHIN API does not resolve him; update HI manually in RosterPage
- [ ] Begin inviting members — work through To Invite tab in RosterPage
- [ ] Monitor invite delivery from tjunker9@gmail.com
- [ ] crack.mp3 audio file (source and place at `public/sounds/crack.mp3`)
- [ ] Delete serviceAccountKey.json from scripts/ after confirming no longer needed
- [ ] Send "coming soon" install instructions email to members
- [ ] Send wave 1 invite emails once members have installed PWA

### Next session priorities:
1. Begin sending member invites through RosterPage To Invite tab
2. Monitor delivery / Safari link issues
3. Phase 3 — Thursday Night Men's League

---

## Platform Roadmap Summary

| Phase | Name | Status | Est. Hours |
|-------|------|--------|-----------|
| 1G | Admin + Deploy | ✅ Complete | — |
| 1H | Club Profile & Platform Foundation | ✅ Complete | — |
| 2A | Multi-Course Support | ✅ Complete | — |
| 2B | Member Doc: tnmlTeam field | ✅ Complete | — |
| 2C | Scheduled Games + UX Fixes | ✅ Complete | — |
| 2D | Member Management | ✅ Complete | — |
| 2E | CSV/Bulk Invite | ✅ Complete | — |
| 2F | Season Filtering / Leaderboard | ✅ Complete | — |
| 2H | Infrastructure, Access Control + Home Page | ✅ Complete | — |
| 2I | GHIN Handicap Sync | ✅ Complete | — |
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

---

## Key Product Decisions (Final)

| ID | Decision |
|----|----------|
| CGAD-001 | Multi-tenant from day one — all data under clubs/{clubId} |
| CGAD-049 | Game edit access restricted to creator + admins |
| CGAD-050 | Home page has no bottom tab bar; Games tab renamed to Home |
| CGAD-051 | GHIN sync is admin-triggered (not scheduled); weekly reminder email Monday 8 AM Central |

Full rationale in DecisionLog.md. Next available ID: **CGAD-052**

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
- GHIN name splitter: `first = parts[0]`, `last = parts.slice(1).join(' ')`
- GHIN sync scope: all members where `inviteStatus != 'skip'`

---

## Session Workflow
1. Load this SessionStarter + upload all 6 docs
2. Also load: BestMethods.md — read relevant section before starting any new feature
3. Design/data model decisions in this session first
4. Code delivered as CC prompts for VS Code Claude
5. Upload files when debugging logic bugs
6. At session end: update all docs
