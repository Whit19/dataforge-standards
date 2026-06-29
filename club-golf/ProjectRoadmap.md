# ProjectRoadmap.md — Club Golf Platform
**Phased build plan — updated June 2026**

---

## Phase Overview

| Phase | Name | Status |
|-------|------|--------|
| Phase 1 | 6-Point Game MVP | ✅ Complete |
| Phase 1H | Club Profile & Platform Foundation | ✅ Complete |
| Phase 2 | Full Member Roster + Auth | ✅ Complete |
| Phase 2I | GHIN Handicap Sync | ✅ Complete |
| Phase 3 | Format Library + EZ Games | 🔲 Not Started |
| Phase 4 | Thursday Night Men's League | 🔲 Not Started |
| Phase 5 | Platform Launch + Monetization | 🔲 Not Started |

---

## Phase 1 — 6-Point Game MVP ✅ Complete

### 1A — Project Setup ✅ COMPLETE
### 1B — Auth + Club Setup ✅ COMPLETE
### 1C — Game Creation ✅ COMPLETE
### 1D — Score Entry ✅ COMPLETE
### 1E — Read-Only Viewer Link ✅ COMPLETE
### 1F — Post-Round + History ✅ COMPLETE

### 1G — Admin + Deploy ✅ COMPLETE
- [x] AdminPage, RosterPage, CoursePage
- [x] Firebase Hosting deploy + VS Code shortcut
- [x] BottomTabBar
- [x] PWA icon — Tripoli windmill
- [x] PWA manifest + apple-touch-icon
- [x] Member invite flow (sendInvite / claimInvite / deleteMember)
- [x] JoinPage (/join)
- [x] "Add to Home Screen" prompt
- [x] Game creation open to all members
- [x] Landscape scorecard view
- [x] ResultsPage UX — back arrow + share, context-aware navigation
- [x] Player picker: modal sheet + favorites
- [x] ProfilePage — stats, favorites, preferences, logout
- [x] HistoryPage — All/Mine toggle
- [x] Firestore security rules — roles-based
- [x] Hole 18 stats bug fixed
- [x] Confirm PWA installable on iOS + Android with testers
- [x] Share with pro / initial club testers
- [x] crack.mp3 audio file

---

## Phase 1H — Club Profile & Platform Foundation ✅ Complete

- [x] Define and populate `clubs/{clubId}/profile` Firestore schema + Tripoli doc
- [x] Build `useClubProfile.js` hook
- [x] Add `clubProfileRef` to `firestorePaths.js`
- [x] GamesPage brand header — "Club Golf — {name}" in gold
- [x] Build `ClubProfilePage` (admin only at /admin/club)
- [x] Consistent page header standard — `.page-title` across all pages
- [x] ProfilePage club name reads from profile doc
- [x] clubId travels explicitly in invite link URL param
- [x] CGAD-028 + CGAD-029 added to DecisionLog

---

## Phase 2 — Full Member Roster + Auth ✅ Complete

### 2A — Multi-Course Support ✅ COMPLETE
### 2B — Member Doc: tnmlTeam Field ✅ COMPLETE
### 2C — Scheduled Games + UX Fixes ✅ COMPLETE
### 2E — CSV/Bulk Invite ✅ COMPLETE

### 2D — Member Management ✅ COMPLETE
- [x] Deactivate member — disables Firebase Auth account, sets inviteStatus: 'skip'
- [x] Reactivate member — re-enables Firebase Auth account, restores inviteStatus: 'accepted'
- [x] deactivateMember + reactivateMember Cloud Functions
- [x] Edit role from RosterPage — reads/writes clubs/{clubId}/roles/{uid}
- [x] Not Active tab restored to RosterPage — tab order: Active | Pending | To Invite | Not Active
- [x] Bulk invite status view — covered by RosterPage filter tabs

### 2F — Season Filtering / Leaderboard ✅ COMPLETE
- [x] Per-year stats buckets on member doc: stats.all (lifetime) + stats[year] (e.g. stats.2026)
- [x] statsUtils.js updated to write to both stats.all and stats[year] on every game save
- [x] migrateStats.cjs migration script — run 5/27/26, migrated 16 members
- [x] LeaderboardPage — new dedicated page at /leaderboard
- [x] BottomTabBar — 5 tabs for all users: Games / History / Leaderboard / Profile / Info
- [x] Admin tab removed from BottomTabBar — Admin Panel button added to ProfilePage (admin only)
- [x] HistoryPage — collapsible Club leaders card removed
- [x] App.jsx — /leaderboard route added
- [x] GamesPage stats strip — reads from stats.all (lifetime)
- [x] CGAD-046 + CGAD-047 added to DecisionLog

### 2G — Data Cleanup + HI Age Indicator ✅ COMPLETE
- [x] updateMemberEmails.cjs — 62 member emails corrected from Golf Genius export
- [x] setDefaultTees.cjs — 159 members set to Blue tees
- [x] backfillHandicapUpdatedAt.cjs — 188 members backfilled from createdAt
- [x] HI Age badge in NewGamePage Players step (green/amber/red, between Handicap and Tees)
- [x] HI Updated column + HI Age sort in RosterPage
- [x] handicapUpdatedAt written in ProfilePage saveHI
- [x] handicapUpdatedAt written in useRoster addMember + updateMember
- [x] handicapUpdatedAt written in NewGamePage auto-fill slot 0
- [x] ProfilePage saveHI plus handicap validation fixed (-10 to 54)
- [x] useRoster addMember null HI fix (no longer defaults to 0)
- [x] CGAD-048 added to DecisionLog

### 2H — Infrastructure, Access Control + Home Page ✅ COMPLETE
- [x] Custom domain — `club-golf.app` live via Firebase Hosting + GoDaddy DNS; SSL cert provisioned 5/31/26
- [x] PWA update banner — `useRegisterSW` hook in App.jsx; "A new version is available / Update Now" banner on SW update
- [x] Invite email overhaul — URL updated to `club-golf.app`, iPhone + Android install instructions added
- [x] Admin Venmo handle edit — venmoHandle field added to RosterPage edit panel + updateMember payload
- [x] exportMembers.cjs — exports active_members.csv + pending_members.csv for mail merge
- [x] ClubGolf_UserGuide.html updated — 5-tab layout, Leaderboard tab, HI Age badge, Starting Hole documented
- [x] Game creator access control — only game creator or admin can edit; all others auto-redirect to read-only viewer (CGAD-049)
- [x] GamesPage redesigned as home page — windmill logo header, stats strip, New Game hero button, Games card (live + scheduled rows), 2×2 nav grid (History / Leaderboard / Profile / Info) (CGAD-050)
- [x] BottomTabBar hidden on home page (/games exact route) — CGAD-050
- [x] Games tab renamed to Home (ti-home icon) — CGAD-050
- [x] CGAD-049 + CGAD-050 added to DecisionLog

### 2I — GHIN Handicap Sync ✅ COMPLETE
- [x] syncHandicaps Cloud Function — onCall, auth required, timeout 180s
- [x] GHIN login → bearer token → search per member by first/last name, WI, Tripoli Country Club
- [x] Name splitter: first = parts[0], last = parts.slice(1).join(' ') — handles compound surnames
- [x] Sync scope: all members where inviteStatus != 'skip'
- [x] Writes handicapIndex + handicapUpdatedAt + hiSyncStatus to each member doc
- [x] Writes lastHiSync summary to clubs/{clubId}/profile/profile doc
- [x] Dry run mode — previews results without writing to Firestore
- [x] sendHiSyncReminder scheduled function — every Monday 13:00 UTC (8 AM Central)
- [x] Admin-only "Sync Handicaps" ghost button on home page below Games card
- [x] Sync modal — credential form → running state → results summary (updated/not found/errors)
- [x] NOT FOUND / ERROR badges in RosterPage HI column
- [x] CGAD-051 added to DecisionLog

---

## Phase 3 — Format Library + EZ Games 🔲 Not Started
**~30–45 hours — largest phase**

### 3A — Format Engine (data-driven)
### 3B — Game Lifecycle Extensions
### 3C — EZ Games (cross-foursome via game code)
### 3D — Foursomes & Groups Page
### 3E — New Format Scoring Engines
### 3F — Side Games Layer
### 3G — Player History per Format

### 3H — Course Import (EZ Course Setup)
- [ ] Downloadable CSV template (hole #, par, stroke index, yardage by tee set)
- [ ] CSV upload in CoursePage — parses template and pre-fills course form
- [ ] Validation + error feedback for malformed rows
- [ ] Stretch goal: golf course API lookup (search by name/location → pre-fill) — evaluate available APIs and coverage before committing

---

## Phase 4 — Thursday Night Men's League 🔲 Not Started
**~12–18 hours**

- [ ] League structure: seasons (date range), weeks, flights
- [ ] League game formats: 6-Point + other formats from Phase 3
- [ ] League leaderboard / season standings
- [ ] Scheduling: week-by-week schedule, tee time assignment
- [ ] Flight management: admin assigns players to flights
- [ ] League history: past season results
- [ ] **Migrate `tnmlTeam` from member doc → `leagues/{leagueId}/roster` entry** (CGAD-035)

---

## Phase 5 — Platform Launch + Monetization 🔲 Not Started
**~20–30 hours**

### 5A — Multi-Club Infrastructure
### 5B — Subscription Billing (Stripe)
### 5C — Multi-Club Admin

---

*Next available Decision ID: CGAD-051*
*Last updated: June 2026 — Session 28 complete. Phase 2H complete. Phase 2 complete. Phase 3/4 resequenced: Format Library ahead of TNML.*
