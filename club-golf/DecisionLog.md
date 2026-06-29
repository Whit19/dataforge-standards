# DecisionLog.md — Club Golf App
**All significant architecture and product decisions with rationale.**

---

## Decision Format
Each entry includes: ID, title, decision made, rationale, date, and status.

---

## Decisions

### CGAD-001 — Multi-Tenant from Day One
- **Decision:** All Firestore data scoped under `clubs/{clubId}/...`.
- **Rationale:** Monetization model requires per-club isolation.
- **Status:** Final

### CGAD-002 — Separate Firebase Project from UP Golf
- **Decision:** Completely isolated Firebase project and React codebase.
- **Rationale:** Different audience, auth requirements, and monetization. Isolation prevents contamination.
- **Status:** Final

### CGAD-003 — Per-Club Subscription Monetization
- **Decision:** Clubs pay ~$50–150/month. No per-player payment in v1.
- **Rationale:** Simplest billing model. Club pro/admin is the customer.
- **Status:** Final

### CGAD-004 — V1 Auth: Active Player Accounts + Read-Only Link
- **Decision:** Score enterers have Firebase Auth accounts. Viewers use read-only link.
- **Rationale:** Minimizes onboarding friction for v1.
- **Status:** Final

### CGAD-005 — Manual Handicap Entry for V1
- **Decision:** Admin or player enters handicap index manually. GHIN API deferred to v2.
- **Status:** Final

### CGAD-006 — Single Course for V1
- **Decision:** One course per club for v1. Multi-course deferred to v2.
- **Status:** Superseded by CGAD-036

### CGAD-007 — Net Scores Drive Points (Birdie Mode Configurable)
- **Decision:** Individual Low Net and Team Low Net always use net scores. Birdie mode configurable (3 options).
- **Status:** Final

### CGAD-008 — Reuse UP Golf UI Patterns
- **Decision:** Score entry, group display, results patterns adapted from UP Golf.
- **Status:** Final

### CGAD-009 — Viewer Link: Direct Share + Today's Games Page
- **Decision:** Share button generates read-only link. Club-level Today's Games page lists active games.
- **Status:** Final

### CGAD-010 — Single Admin Role for V1
- **Decision:** One admin role per club. No distinction between club admin and game admin in v1.
- **Status:** Final

### CGAD-011 — Pickable Roster with Guest Player Support
- **Decision:** Persistent roster + per-game guest players. Guests not saved to roster.
- **Status:** Final

### CGAD-012 — Per-Hole Stroke Index Required
- **Decision:** Course setup requires per-hole stroke index (1–18). Used for net score calculation.
- **Status:** Final

### CGAD-013 — CRACK Setting Options (Revised)
- **Decision:** Three CRACK options: Manual Only, Auto Crack, Auto Re-Crack. Configurable holes per game. Mutually exclusive modes.
- **Date:** 4/29/26
- **Status:** Final

### CGAD-014 — Birdie Mode: 3 Options
- **Decision:** Gross Birdie / Net Birdie + Basket / Net Birdie No Basket.
- **Date:** 4/29/26
- **Status:** Final

### CGAD-015 — Configurable Auto-Crack Holes Per Game
- **Decision:** Auto crack holes stored per game as arrays. Any hole 1–18. Default [9, 18].
- **Date:** 4/29/26
- **Status:** Final

### CGAD-016 — Casket
- **Decision:** Optional. Both players gross birdie + sweep = 24 pts. Crack multiplier applies on top.
- **Date:** 4/29/26
- **Status:** Final

### CGAD-017 — Bonus Points: Sandy, Pully, Manual
- **Decision:** Three optional bonus types, all 1 pt, all manual entry. Isolated from basket/casket. (raw + bonus) × multiplier = total.
- **Date:** 4/29/26
- **Status:** Final

### CGAD-018 — Birdie Point: Best Score Wins When Both Teams Have Birdie-or-Better
- **Decision:** Lower score wins birdie point. Eagle beats birdie. Gross beats net at same score. True tie cancels.
- **Date:** 4/29/26
- **Status:** Final

### CGAD-019 — Per-Member Stats: 9 Fields, Gross Birdies Only
- **Decision:** On game complete, statsUtils.js writes 9 increment fields per member. Guest players skipped. Non-blocking.
- **Date:** 5/5/26
- **Status:** Final

### CGAD-020 — Member Invite Flow
- **Decision:** Admin pre-creates roster entries. Members claim their account via a tokenized email invite link. Flow: admin sends invite from RosterPage → Cloud Function emails branded link → member lands on /join → sets password → Firebase Auth account created → UID linked to roster doc → auto sign-in → redirected to /games. Token is single-use, 7-day expiry. Deleting a member removes both Firestore doc and Firebase Auth account via deleteMember Cloud Function.
- **Date:** 5/7/26
- **Status:** Final

### CGAD-021 — All Authenticated Members Can Create Games
- **Decision:** The "+ New Game" button and game creation flow are available to all authenticated members, not admin-only.
- **Date:** 5/7/26
- **Status:** Final

### CGAD-022 — Scorecard View: Orientation-Driven Layout
- **Decision:** Rotating the phone to landscape shows a traditional golf scorecard. Portrait shows points/hole view. Available on ScoreEntryPage, ResultsPage, ViewerPage. No tab switcher.
- **Date:** 5/8/26
- **Status:** Final

### CGAD-023 — Default Crack Setting: AUTO_RECRACK
- **Decision:** New game creation defaults `crackSetting` to `CRACK_SETTING.AUTO_RECRACK`.
- **Date:** 5/11/26
- **Status:** Final

### CGAD-024 — Casket Enabled by Default
- **Decision:** New game creation defaults `casketEnabled` to `true`.
- **Date:** 5/11/26
- **Status:** Final

### CGAD-025 — Leaderboard in HistoryPage as Collapsible Card
- **Decision:** Club leaderboard lives as a collapsible card at the top of HistoryPage, not a separate tab or page.
- **Date:** 5/12/26
- **Status:** Superseded by CGAD-047

### CGAD-026 — Points Leaderboard Sorts by Differential
- **Decision:** The Points tab sorts and displays by points differential (pointsFor − pointsAgainst), not raw pointsFor.
- **Date:** 5/12/26
- **Status:** Final

### CGAD-027 — Favorites Stored on Member Doc
- **Decision:** `favorites: []` array of memberIds stored on the current user's member doc. Managed from ProfilePage and RosterPage. Surfaced in player picker modal.
- **Date:** 5/12/26
- **Status:** Final

### CGAD-028 — Club Profile Doc is Source of Truth for Club Name/Branding
- **Decision:** `clubs/{clubId}/profile` is the authoritative source for club name and branding. Never hardcode club name anywhere in the app.
- **Date:** 5/13/26
- **Status:** Final

### CGAD-029 — App Header Displays "Club Golf — {Club Name}"
- **Decision:** GamesPage brand header shows "Club Golf — {name}" from the club profile doc. Falls back to "Club Golf" while loading.
- **Date:** 5/13/26
- **Status:** Final

### CGAD-030 — Player-First App
- **Decision:** No pro shop control features. Player experience is the primary design target.
- **Status:** Final

### CGAD-031 — One Primary Format + Side Games Only
- **Decision:** No dual primary formats. One primary + any number of opt-in side games.
- **Status:** Final

### CGAD-032 — Scheduled → Active → Complete State Machine
- **Decision:** All games follow a three-state lifecycle. `GAME_STATUS` values: `scheduled`, `active`, `complete`. Games created with a future date are always scheduled. Games created for today offer Start Now or Save for Later.
- **Date:** 5/14/26
- **Status:** Final

### CGAD-033 — Hybrid Monetization
- **Decision:** Free tier + Pro subscription (~$50–75/month) + per-event option (~$10–25).
- **Status:** Pending Phase 5

### CGAD-034 — Guests: Participate, No Account, No History, Not Billed
- **Decision:** Guest players identified by name only. No Firebase Auth account. Not counted in billing.
- **Status:** Final

### CGAD-035 — tnmlTeam Field on Member Doc (Temporary)
- **Decision:** Add `tnmlTeam: string | null` as a free-text field on the member doc. Admin-editable from RosterPage edit panel. Explicit migration target: Phase 3 league roster structure (`leagues/{leagueId}/roster` entry).
- **Date:** 5/13/26
- **Status:** Final (temporary — migrates in Phase 3)

### CGAD-036 — Multi-Course Support (Manual Entry)
- **Decision:** CoursePage is now a course list view with per-course edit. `useCourseList(clubId)` returns all courses. `useCourse(clubId, courseId?)` loads a specific course by ID, falls back to first if courseId omitted. NewGamePage auto-selects if only one course exists; shows dropdown if two or more. No course API integration in Phase 2.
- **Date:** 5/13/26
- **Status:** Final

### CGAD-037 — Date-Driven Game Scheduling
- **Decision:** Game date is set in the Players step of NewGamePage (defaults to today). If date > today, the Review step shows a single "Save Game" button — the game is inherently scheduled. If date = today, two buttons are shown: "Start Now" and "Save for Later". No explicit "Schedule for Later" toggle — the date drives the behavior.
- **Date:** 5/14/26
- **Status:** Final

### CGAD-038 — Start Round Gated on Foursome Membership or Admin Role
- **Decision:** The "Start Round" button on a scheduled game card is only visible to players whose memberId is in the game's `playerIds` array, or to admins. For shell games with no players set, the button is visible to admins only.
- **Date:** 5/14/26
- **Status:** Final

### CGAD-039 — Plus Handicaps Entered as Negative Numbers
- **Decision:** Plus handicaps (better than scratch) are entered as negative numbers throughout the app (e.g. -3.5 for a +3.5 handicap index). Valid range is -10 to 54.
- **Date:** 5/14/26
- **Status:** Final

### CGAD-040 — inviteStatus Values Define Roster Filter Buckets
- **Decision:** `inviteStatus` on member doc drives four roster filter tabs: `null` = To Invite; `'pending'` = invite sent, not yet claimed; `'accepted'` = account created and active; `'skip'` = deactivated or not flagged for inviting (Not Active). Tab order: Active | Pending | To Invite | Not Active.
- **Date:** 5/15/26
- **Status:** Final

### CGAD-041 — Blank Handicap Index Stored as null, Never 0
- **Decision:** When a member's handicap index is not known or not entered, it is stored as `null` — never defaulted to `0`.
- **Date:** 5/15/26
- **Status:** Final

### CGAD-042 — Stats Only Written on Save Path
- **Decision:** `updateMemberStats` is only called when an organizer chooses "Save" after completing a round. Choosing "Discard" deletes the game doc only — no stats are written.
- **Date:** 5/15/26
- **Status:** Final

### CGAD-043 — GamesPage redesigned as single scrolling feed
- **Decision:** GamesPage is the app home page. Layout: windmill logo header, stats strip, New Game hero button, Games card (live + scheduled rows inline), 2×2 nav grid. Superseded feed layout replaced entirely.
- **Date:** 5/18/26
- **Status:** Final

### CGAD-044 — Venmo handle stored on member doc, shown on ResultsPage
- **Decision:** Members enter their own Venmo handle in ProfilePage. On ResultsPage, a 💸 link appears next to any player who has one set.
- **Date:** 5/18/26
- **Status:** Final

### CGAD-045 — Starting Hole + numHoles stored in game settings
- **Decision:** Two new fields added to game settings: `startingHole` (integer 1–18, default 1) and `numHoles` (9 or 18, default 18). The hole sequence is computed in ScoreEntryPage and never stored.
- **Date:** 5/26/26
- **Status:** Final

### CGAD-046 — Per-Year Stats Buckets on Member Doc
- **Decision:** Member stats are stored as a nested object: `stats.all` holds lifetime aggregates; `stats[year]` (e.g. `stats.2026`) holds per-year aggregates. Both are updated atomically on every game save. Year is derived from `game.date.slice(0, 4)`. GamesPage stats strip reads from `stats.all`. LeaderboardPage reads from the selected year bucket or `stats.all` for All Time. Migration script (migrateStats.cjs) run 5/27/26 to migrate 16 existing members.
- **Date:** 5/27/26
- **Status:** Final

### CGAD-047 — Leaderboard as Dedicated Tab; 5 Tabs for All Users
- **Decision:** Leaderboard moved from a collapsible card in HistoryPage to a dedicated LeaderboardPage at `/leaderboard`. BottomTabBar is 5 tabs for all users: Games / History / Leaderboard / Profile / Info. Admin tab removed from BottomTabBar — admins access the Admin Panel via a button in ProfilePage.
- **Date:** 5/27/26
- **Status:** Final

### CGAD-048 — HI Age Indicator: Color-Coded Badge at Game Creation
- **Decision:** `handicapUpdatedAt` Firestore Timestamp is written on every HI save (ProfilePage, useRoster addMember/updateMember) and read in NewGamePage and RosterPage. A color-coded HI Age badge appears between Handicap and Tees in the NewGamePage Players step: green = <15 days, amber = 15–30 days, red = 30+ days or never set. RosterPage shows an HI Updated column with the same coding, plus an "HI Age" sort option (oldest first). Bulk-imported members were backfilled using `createdAt` as a stand-in via backfillHandicapUpdatedAt.cjs (run 5/29/26).
- **Rationale:** No dependency on pro shop export schedule. Surfaces stale handicaps to the organizer at the moment they matter — game setup — and lets players self-correct via ProfilePage.
- **Date:** 5/29/26
- **Status:** Final

### CGAD-049 — Game Edit Access Restricted to Creator + Admins
- **Decision:** Only the game creator (`game.createdBy === user.uid`) or an admin (`isAdmin`) can access ScoreEntryPage in edit mode. All other authenticated users who navigate to `/games/:gameId` are automatically redirected to the read-only viewer at `/view/:gameId?token=:readOnlyToken` via a useEffect access gate in ScoreEntryPage.jsx.
- **Rationale:** Real-world usage pattern is one person runs the scoring session. No collaborative score entry needed in v1. Prevents accidental or unauthorized edits.
- **Alternatives considered:** Disabling inputs for non-creators (Option B) — rejected as more fragile and more code.
- **Date:** 6/10/26
- **Status:** Final

### CGAD-050 — Home Page: No Bottom Nav; Games Tab Renamed to Home
- **Decision:** The bottom tab bar is hidden on the home page (/games exact route). On all other pages the tab bar shows with "Games" renamed to "Home" (ti-home icon), pointing to /games. This gives the home page a clean full-screen layout while preserving tab navigation on every inner page.
- **Rationale:** The home page has its own 2×2 nav grid making the tab bar redundant and visually cluttered. All inner pages still need one-tap access back to home.
- **Alternatives considered:** Keeping the tab bar on home with a selected "Home" state — rejected as redundant with the nav grid.
- **Date:** 6/11/26
- **Status:** Final

### CGAD-051 — GHIN Handicap Sync: Admin-Triggered with Weekly Reminder
- **Decision:** HI sync is admin-triggered (not fully automated). Admin clicks "Sync Handicaps" on home page, enters GHIN credentials in a modal, Cloud Function fetches HIs for all non-deactivated members. A scheduled Cloud Function emails tjunker9@gmail.com every Monday at 8 AM Central as a reminder. Credentials are never stored — passed at call time only.
- **Rationale:** No free GHIN account available for automation credentials. Admin-triggered avoids credential storage risk while keeping the flow to a single button tap. Weekly reminder preserves the "automatic feel" without the security tradeoff.
- **Alternatives considered:** Fully automated scheduled sync with stored credentials — rejected due to credential storage risk and no throwaway GHIN account available.
- **Date:** 6/18/26
- **Status:** Final

---

## Decision ID Sequence
Next available ID: **CGAD-052**
