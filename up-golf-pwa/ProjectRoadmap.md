# ProjectRoadmap.md — UP Golf PWA
**Phased build plan with milestones and completion tracking**

---

## Phase Overview

| Phase | Name | Status |
|-------|------|--------|
| Phase 0 | Project Setup & Documentation | ✅ Complete |
| Phase 1 | PWA Foundation & Authentication | ✅ Complete |
| Phase 2 | Scoring Engine & Leaderboard | ✅ Complete |
| Phase 3 | Prize Pool, Skins & Payments | ✅ Complete |
| Phase 4 | Chat, Media & Gallery | ✅ Complete |
| Phase 4.5 | Screen Layout Polish + Chat Fixes | ✅ Complete |
| Phase 5 | Push Notifications & Offline Sync | ✅ Complete |
| Phase 6 | Admin Panel (Full) | ✅ Complete |
| Phase 7 | Polish, Testing & Deployment | ✅ Complete |
| Phase 8 | Multi-Tournament & History | 🟡 In Progress |

---

## Phase 7 — ✅ COMPLETE

### Completed
- [x] Header standardization across all screens
- [x] ResizeObserver + offsetHeight pattern on Foursomes, Leaderboard, AdminPanel
- [x] Scores nav icon — landscape leaderboard SVG
- [x] Leaderboard header icon updated to match nav icon
- [x] Bottom nav Home icon, Groups icon custom SVG
- [x] Chat tabs → seg-control buttons
- [x] Info/Money/Admin header fixes
- [x] Player activation tracking: lastLoginAt + AdminPlayers banner + badge
- [x] Push notifications foreground fix
- [x] Birdie notify deduplication guard
- [x] GCP paid account activated
- [x] Dashboard tee times — pill display
- [x] TournamentSummaryCard — field name fixes, age group fixes, gold borders, alignment
- [x] Leaderboard live +/- — calculated against played par only
- [x] Overall leaderboard — shows +/- per round and total
- [x] Match cards — "Up X" only, gold border on winning team
- [x] R3 tee times — generatePairings assigns tee times; Foursomes.jsx lookup
- [x] MoneyScreen — defaults to last locked round
- [x] AdminPayments — R2/R3/R4 paid count badges; crash fix
- [x] R3 scramble prizes — root cause fixed (playerIds + holes exposed)
- [x] Info/Rules — Firestore-backed with admin inline edit UI
- [x] prizes.js: notifyRoundLocked moved above idempotency guard
- [x] usePushNotifications.js: checkAndRefreshToken on every mount
- [x] Push notification fixes — full end-to-end validation ✅
- [x] Admin broadcast push UI (AdminPanel home screen)
- [x] Firestore persistence API migration
- [x] Firestore path cleanup — ALL screens and hooks complete
- [x] Lighthouse audit: Performance 85, Accessibility 95, Best Practices 100 ✅
- [x] Accessibility fixes: viewport meta, main landmark, login contrast
- [x] Offline cold-start fix — useAuth.jsx setLoading unblocked from loadProfile
- [x] ScoreEntry sync status — driven by real isOnline state
- [x] Push prompt flash fix — isSupportChecked gates isInitialising
- [x] firebase.json — index.html no-cache header added
- [x] Notification re-enable bell badge/button in Dashboard welcome card
- [x] Cross-browser testing: Safari iOS ✅, Chrome Android ✅, Chrome Desktop ✅
- [x] PWA install testing on iPhone ✅
- [x] Player Setup Guide (Word doc)
- [x] Leaderboard sort by score-to-par (all tabs); age filter on all round tabs; medal fix; color fix
- [x] User credential migration — all 32 accounts updated to FirstnameLastname format
- [x] Danny Sanicki Auth account resolved
- [x] Admin score entry for any group — isAdmin bypass + group picker in ScoreEntry.jsx ✅
- [x] Duplicate push notifications fixed — native push event handler in SW (DEC-115) ✅
- [x] FCM token accumulation fixed — array replace not append (DEC-116) ✅
- [x] birdieNotify.js data.url added for click navigation (DEC-117) ✅
- [x] firestorePaths.js foursome doc ID — added `G` prefix (ISS-114) ✅
- [x] firestorePaths.js match doc ID — added `M` prefix (ISS-115) ✅
- [x] AdminFoursomes — Team 1/VS/Team 2 UI; saves match docs in sync with foursomes (ISS-116) ✅
- [x] displayName flash on load fixed (ISS-118) ✅
- [x] Skins opt-in cutoff — closes on round lock only (ISS-117) — deployed, pending dry run verify
- [x] Test data tool R3 delete — now also removes foursomes collection docs (ISS-119) ✅
- [x] Scramble team name — admin-only via AdminFoursomes; TeamNameEditor removed (ISS-124) ✅
- [x] Team name displayed in Leaderboard R3, MoneyScreen, Dashboard tee time card ✅
- [x] R3 Leaderboard sort fixed — derived from scrambleResults with full tiebreaker (ISS-120) ✅
- [x] Leaderboard defaults to active round on load (ISS-121) ✅
- [x] MoneyScreen defaults to active round on load (ISS-122) ✅
- [x] Tiebreaker display in R3Tab, RoundTab, ScrambleResultsList (ISS-123) ✅
- [x] Dashboard round format text — dynamic via FORMAT_LABELS (ISS-125, DEC-125) ✅
- [x] Complete Round button + confirmation in ScoreEntry; completedAt on foursome doc (ISS-126, DEC-126) ✅
- [x] Dashboard Round Complete card — replaces tee time card when round done (ISS-127) ✅
- [x] MoneyScreen winnings gated on round lock (ISS-128, DEC-127) ✅
- [x] Dashboard My Winnings gated on round lock (ISS-129, DEC-127) ✅
- [x] Font consistency — leaderboard numbers, admin mono strings normalised (ISS-130, DEC-128) ✅
- [x] Chat scroll to bottom on load and new messages (ISS-131, DEC-129) ✅
- [x] Foursomes R2/R4 holes-thru indicator on match cards (ISS-132, DEC-130) ✅
- [x] Three-state round system: Scheduled → Scoring → Locked (DEC-131, DEC-132) ✅
- [x] Dashboard player view: three states tied to round state (DEC-133, ISS-133) ✅
- [x] Tournament Complete decoupled from R4 lock — manual admin button (DEC-134) ✅
- [x] Tournament status card title dynamic from round state (DEC-135) ✅
- [x] bulkCreatePlayers FieldValue bug fixed (ISS-134) ✅
- [x] ageCategory field cleanup — all 32 player docs use ageGroup only (ISS-135) ✅
- [x] Dry run complete — R1–R4 with 8-player group ✅
- [x] Offline functionality testing — Android ✅
- [x] In-app PDF guides (App Setup + User Guide) added to Info screen
- [x] Player credential distribution to all 32 players complete
- [x] Admin walkthrough of round controls and Tournament Lock complete
- [x] ISS-117 skins opt-in verified
- [x] ISS-136 / DEC-136 — "Add Dining Event" Save button unreachable on iOS PWA fixed; AnnouncementModal/DiningEventModal → inline forms (post-freeze exception)
- [x] Player Profile card on Info screen — selfie upload, phone, casino game, years played (DEC-137) ✅
- [x] Players Directory screen (/players) — 2-col tile grid, sort controls, tap-to-call (DEC-138) ✅
- [x] Firebase Storage rules updated — profiles/{uid}/ path added ✅
- [x] firstYear field set on all 32 player Firestore docs via bulk script ✅
- [x] ProfileEditModal sticky footer — buttons always visible above bottom nav ✅
- [x] Post-freeze fix: payments Firestore read rule opened to all signed-in players — fixes players unable to see live Skins standings on Money screen (ISS-140, DEC-139) ✅

---

## Phase 8 — Multi-Tournament, Net Scoring & Flexible Formats 🟡 In Progress

### Completed in Phase 8 (2026-06-24)
- [x] Tournament Results PDF export — `TournamentResultsPDF.jsx` at `/results-pdf` (admin) and `/history/:year` (player) (DEC-140) ✅
- [x] History entry point — 🏆 History card in Info screen navigates to `/history/2026` (DEC-141) ✅
- [x] Historical data Excel — `UP_Golf_Historical_Import_v3.xlsx` audited and finalized (DEC-142) ✅
  - 6 sheets: Tournaments, Players, Courses, Rounds, Results, Standings
  - All 14 years (2011–2025) populated; ranks/payouts/formats pre-filled
  - Slope/rating pre-filled for all 5 UP Golf courses
- [x] Session docs moved to public dataforge-standards GitHub repo (DEC-144) ✅
- [x] CC prompts now always delivered as downloadable MD files (DEC-143) ✅

### 8A — Multi-Tournament Architecture (Next Priority)
- [ ] Historical data import script — Node.js, reads UP_Golf_Historical_Import_v3.xlsx,
      writes to `historicalTournaments`, `historicalResults`, `historicalStandings`,
      `historicalPlayers`, `courses` Firestore collections
- [ ] History browser — `/history` route lists all years from `historicalTournaments`;
      tapping year navigates to `/history/:year` (route already exists)
- [ ] `config/active` Firestore document → `useActiveTournament.js` hook (replaces `ACTIVE_OUTING_ID` constant)
- [ ] `tournaments` collection (one doc per tournament, replacing `outings`)
- [ ] `enrollments` collection (links players to tournaments)
- [ ] Admin: Tournament List screen
- [ ] Admin: Create Tournament wizard (name, dates, format config, buy-in, player enrollment)
- [ ] Admin: Set Active Tournament toggle
- [ ] Admin: Clone Tournament (copy format config to new tournament)
- [ ] Career stats per player (across all tournaments)

### 8B — Course Data Integration
- [ ] Integrate GolfCourseAPI.com (free) or GolfAPI.io (paid, 42k+ courses) for course search
- [ ] Course search UI in AdminCourseLibrary
- [ ] Extended `courses` Firestore collection schema with per-hole strokeIndex
- [ ] One-time API fetch at setup; stored in Firestore

### 8C — Handicap & Net Scoring Engine
- [ ] `handicapIndex` field on `users` doc (admin-entered)
- [ ] Course Handicap calculation: `HI × (Slope / 113) + (CourseRating - Par)`
- [ ] Net score per hole = `gross - strokesReceived`
- [ ] `useNetScoring.js` hook

### 8D — Flexible Game Format Engine
- [ ] Format definitions stored in Firestore `tournaments/{id}.roundFormats[]`
- [ ] Supported format types: `individual_gross`, `individual_net`, `best_ball_net`, `variable_best_ball_net`, `scramble`, `stableford_gross`, `stableford_net`
- [ ] Leaderboard engine extended to handle all format types
- [ ] Skins available in gross or net mode

### 8E — Admin & UX
- [ ] Per-player handicap index entry in AdminPlayers
- [ ] Round Setup UI shows computed Course Handicap
- [ ] Scorecard entry shows strokes-received indicators per hole

### Phase 8 Key Decisions (Pre-Recorded)
- **DEC-101** — Handicap Index is admin-entered, not pulled from GHIN API
- **DEC-102** — Course data fetched once at admin setup time; stored in Firestore
- **DEC-103** — Format engine is data-driven (Firestore config), never hardcoded to round numbers
- **DEC-104** — `variable_best_ball_net` supports arbitrary `{ par3, par4, par5 }` counting score maps
- **DEC-105** — `useActiveTournament.js` hook replaces `ACTIVE_OUTING_ID` constant
