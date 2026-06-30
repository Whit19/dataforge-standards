# DecisionLog.md — UP Golf PWA
**Running log of all significant design and technical decisions**

---

## DEC-001 — DEC-071
*(See previous session docs — unchanged)*

---

## DEC-072 — Round Lock as Sole Source of Truth for Tournament Status
**Date:** 2026-03-31 | **Category:** Architecture | **Status:** Final

---

## DEC-073 — Admin Navigation: Bottom Nav Always Visible
**Date:** 2026-03-31 | **Category:** UX / Architecture | **Status:** Final

---

## DEC-074 — AdminFoursomes Editable on All Rounds
**Date:** 2026-03-31 | **Category:** UX / Architecture | **Status:** Final

---

## DEC-075 — SVG Lock Icons Replace Emoji in Admin Controls
**Date:** 2026-03-31 | **Category:** Technical / UX | **Status:** Final

---

## DEC-076 — Tournament Summary Card Replaces Status Card on Completion
**Date:** 2026-03-31 | **Category:** UX | **Status:** Final

---

## DEC-077 — Dynamic Fixed Header Height via ResizeObserver + offsetHeight
**Date:** 2026-03-31 | **Category:** Technical | **Status:** Final

---

## DEC-078 — Push Notification Foreground Handling
**Date:** 2026-03-31 | **Category:** Technical | **Status:** Superseded by DEC-115

---

## DEC-079 — Scores Nav Icon: Landscape Leaderboard Scoreboard SVG
**Date:** 2026-04-13 | **Category:** UX | **Status:** Final

---

## DEC-080 — Player Activation Tracking
**Date:** 2026-03-31 | **Category:** Feature | **Status:** Final

---

## DEC-081 — Groups Icon: SVG Stick Figure Pair
**Date:** 2026-04-01 | **Category:** UX | **Status:** Final

---

## DEC-082 — Match Status Display: Always "Up X", Never "Down X"
**Date:** 2026-04-01 | **Category:** UX | **Status:** Final

---

## DEC-083 — Rules Tab: Firestore-Backed with Admin Edit UI
**Date:** 2026-04-01 | **Category:** Architecture | **Status:** Final

---

## DEC-084 — Overall Leaderboard Shows +/- Not Raw Gross
**Date:** 2026-04-01 | **Category:** UX | **Status:** Final

---

## DEC-085 — scrambleResults Shape in useLeaderboard
**Date:** 2026-04-01 | **Category:** Technical | **Status:** Final

---

## DEC-086 — R3 Tee Times: Assigned by Team Index at Generation Time
**Date:** 2026-04-01 | **Category:** Architecture | **Status:** Final

---

## DEC-087 — Push Notification: notifyRoundLocked Above Idempotency Guard
**Date:** 2026-04-02 | **Category:** Technical | **Status:** Final

---

## DEC-088 — Push Foreground Listener in Standalone useEffect
**Date:** 2026-04-02 | **Category:** Technical | **Status:** Superseded by DEC-115

---

## DEC-089 — Silent Token Refresh on Every Mount
**Date:** 2026-04-02 | **Category:** Technical | **Status:** Final

---

## DEC-090 — checkAndRefreshToken Never Writes fcmTokens:[] to Firestore
**Date:** 2026-04-02 | **Category:** Technical | **Status:** Superseded by DEC-116

---

## DEC-091 — generatePairings Delays Push 2 Seconds After batch.commit()
**Date:** 2026-04-02 | **Category:** Technical | **Status:** Final

---

## DEC-092 — onBackgroundMessage Foreground Guard via clients.matchAll()
**Date:** 2026-04-02 | **Category:** Technical | **Status:** Superseded by DEC-115

---

## DEC-093 — Admin Broadcast Push UI
**Date:** 2026-04-02 | **Category:** Feature | **Status:** Final

---

## DEC-094 — Firestore Offline Cache API Migration
**Date:** 2026-04-02 | **Category:** Technical | **Status:** Final

---

## DEC-095 — COLLECTIONS.RULES Added to constants.js
**Date:** 2026-04-02 | **Category:** Technical | **Status:** Final

---

## DEC-096 — Firestore Path Cleanup Complete Across All Files
**Date:** 2026-04-02 | **Category:** Technical | **Status:** Final

---

## DEC-097 — Viewport Meta: user-scalable Removed for Accessibility
**Date:** 2026-04-02 | **Category:** Technical / Accessibility | **Status:** Final

---

## DEC-098 — main Landmark for Accessibility
**Date:** 2026-04-02 | **Category:** Accessibility | **Status:** Final

---

## DEC-099 — Login Screen Text Contrast Fix
**Date:** 2026-04-02 | **Category:** Accessibility | **Status:** Final

---

## DEC-100 — Lighthouse PWA Category No Longer Applicable
**Date:** 2026-04-02 | **Category:** Process | **Status:** Final

---

## DEC-101 — Phase 8: Handicap Index is Admin-Entered, Not GHIN-Connected
**Date:** 2026-04-04 | **Category:** Architecture | **Status:** Final

---

## DEC-102 — Phase 8: Course Data Fetched Once at Setup via External API
**Date:** 2026-04-04 | **Category:** Architecture | **Status:** Final

---

## DEC-103 — Phase 8: Format Engine is Data-Driven, Never Hardcoded
**Date:** 2026-04-04 | **Category:** Architecture | **Status:** Final

---

## DEC-104 — Phase 8: variable_best_ball_net Uses Par-Keyed countingScores Object
**Date:** 2026-04-04 | **Category:** Architecture | **Status:** Final

---

## DEC-105 — Phase 8: useActiveTournament Hook Replaces ACTIVE_OUTING_ID Constant
**Date:** 2026-04-04 | **Category:** Architecture | **Status:** Final

---

## DEC-106 — Notification Re-enable Bell Badge in Dashboard Welcome Card
**Date:** 2026-04-13 | **Category:** Feature / UX | **Status:** Final

---

## DEC-107 — useAuth Offline Cold-Start Fix: setLoading Unblocked
**Date:** 2026-04-13 | **Category:** Technical | **Status:** Final

---

## DEC-108 — ScoreEntry Sync Status Driven by Real Network State
**Date:** 2026-04-13 | **Category:** Technical | **Status:** Final

---

## DEC-109 — Push Prompt Flash Fix: isSupportChecked Gates isInitialising
**Date:** 2026-04-13 | **Category:** Technical | **Status:** Final

---

## DEC-110 — firebase.json: index.html Served with no-cache
**Date:** 2026-04-13 | **Category:** Technical | **Status:** Final

---

## DEC-111 — birdieNotify.js Has Independent Send Logic from push.js
**Date:** 2026-04-13 | **Category:** Technical | **Status:** Final

---

## DEC-112 — FCM Token Accumulation: arrayUnion Never Removes Old Device Tokens
**Date:** 2026-04-13 | **Category:** Technical | **Status:** Superseded by DEC-116

---

## DEC-113 — Admin Score Entry Bypass: Group Picker in ScoreEntry.jsx
**Date:** 2026-04-14 | **Category:** Feature / UX | **Status:** Final

---

## DEC-114 — Player Username Format: Full FirstnameLastname
**Date:** 2026-04-14 | **Category:** Architecture | **Status:** Final

---

## DEC-115 — Push Notifications: Native push Event Replaces Firebase Compat SDK in SW
**Date:** 2026-04-21 | **Category:** Technical | **Status:** Final

---

## DEC-116 — FCM Token Storage: Replace Array, Never Append
**Date:** 2026-04-21 | **Category:** Technical | **Status:** Final

---

## DEC-117 — birdieNotify.js: data.url Added to FCM Payload
**Date:** 2026-04-21 | **Category:** Technical | **Status:** Final

---

## DEC-118 — AdminFoursomes: Team Split UI + Match Doc Sync on Save
**Date:** 2026-04-22 | **Category:** Feature / Architecture | **Status:** Final

---

## DEC-119 — Firestore Doc ID Prefixes: G for Foursomes, M for Matches
**Date:** 2026-04-22 | **Category:** Architecture | **Status:** Final

---

## DEC-120 — Skins Opt-In Cutoff: Round Lock Only
**Date:** 2026-04-22 | **Category:** UX | **Status:** Final

---

## DEC-121 — Scramble Team Names: Admin-Only via AdminFoursomes
**Date:** 2026-04-23 | **Category:** UX / Architecture | **Status:** Final
**Decision:** Removed player-facing `TeamNameEditor` component from `Foursomes.jsx`. Team names are set exclusively by admins in AdminFoursomes. Player-facing entry had multiple issues and is not worth maintaining. `teamName` field remains on `scrambleTeams` docs.

---

## DEC-122 — r3Leaderboard Derived from scrambleResults
**Date:** 2026-04-23 | **Category:** Technical | **Status:** Final
**Decision:** `r3Leaderboard` useMemo now derives from `scrambleResults` rather than sorting `r3Scores` independently. This ensures the Leaderboard R3 tab and MoneyScreen scramble results always show identical ranking. `rank` uses `s.place` (tiebreaker-aware) instead of array index. `scrambleResults` must be declared before `r3Leaderboard` in `useLeaderboard.js` — wrong order causes ReferenceError.

---

## DEC-123 — Default Tab to Active Round: Leaderboard + MoneyScreen
**Date:** 2026-04-23 | **Category:** UX | **Status:** Final
**Decision:** Both `LeaderboardScreen` and `MoneyScreen` now default to the currently active round on load. `LeaderboardScreen` reads `outing.status` and maps to tab key (`round1`→`r1` etc.). `MoneyScreen` uses last locked round among 2/3/4 (default 2 if none locked). Both use a `hasDefaulted`/`hasDefaultedWin` ref so the default fires once only and doesn't override user tab changes mid-session. `hasDefaultedSkins` added for Skins tab.

---

## DEC-124 — Tiebreaker Scores Displayed in Leaderboard and MoneyScreen
**Date:** 2026-04-23 | **Category:** UX | **Status:** Final
**Decision:** Tiebreaker detail shown whenever two or more players/teams share the same gross score (`.some()` check, not `.every()`). Labels: `Back 9 Score: ##` or `Last 3 Holes Score: ##`. Applies to R3 tab (scramble teams), R1/R2/R4 RoundTab, and ScrambleResultsList in MoneyScreen. Shown for ALL tied entries, not just winner. Required attaching `backNine` and `lastThree` to R1/R2/R4 row objects. R1 rows attach these directly (R1 bypasses `sortWithTiebreaker`).

---

## DEC-125 — Dashboard Round Format Text: Dynamic via FORMAT_LABELS
**Date:** 2026-04-24 | **Category:** UX / Architecture | **Status:** Final
**Decision:** `STATUS_CONFIG` no longer hardcodes format description text for round1–round4 entries. Text is derived in JSX from `FORMAT_LABELS[activeRound?.format]`, falling back to `cfg.text` for setup/complete statuses. Prevents drift if round format is changed in Firestore (e.g. R4 was showing "Combined gross 2v2" instead of "Best Ball 2v2").

---

## DEC-126 — Complete Round: Player Confirmation + completedAt on Foursome Doc
**Date:** 2026-04-24 | **Category:** Feature / Architecture | **Status:** Final
**Decision:** "Complete Round" button appears in `ScoreEntry.jsx` when all 18 holes are entered for all players (or round is already locked). Player taps button → confirmation panel shows per-player gross/to-par summary → on confirm, `completedAt: serverTimestamp()` written to foursome doc. Dashboard reads `myFoursome.completedAt != null || activeRound.isLocked` to show Round Complete card. Rainout/early-lock handled cleanly: admin locks round → `isLocked` drives complete state regardless of holes played. `completedAt` field preserved for future use.

---

## DEC-127 — Winnings Gated on Round Lock in Dashboard and MoneyScreen
**Date:** 2026-04-24 | **Category:** UX | **Status:** Final
**Decision:** Winnings are only shown once the corresponding round is locked by admin. In Dashboard `myWinnings` useMemo: R2 match winnings gated on R2 locked, scramble on R3 locked, R4 match on R4 locked, skins per-round lock. In MoneyScreen `WinningsTab`: results replaced with "Results will appear once Round X is locked" placeholder until round is locked. Prevents showing live/changing winnings mid-round.

---

## DEC-128 — Font Consistency: UI Font for All Leaderboard Numbers
**Date:** 2026-04-24 | **Category:** UX / Technical | **Status:** Final
**Decision:** Removed `font-family: var(--font-mono)` from `.lb-table__seed-num`, `.lb-table__score`, `.lb-table__tpar` in `index.css`. All leaderboard numbers now use `var(--font-ui)` for visual consistency. Admin screens that display code-like data (`AdminBulkImport`, `AdminRoundSetup`, `AdminPlayers`) use `var(--font-mono)` explicitly. Bare `'monospace'` strings replaced with `var(--font-mono)`. Redundant `fontFamily: 'inherit'` removed from `AdminPayments`, `AdminPanel`, `InfoScreen`.

---

## DEC-129 — Chat Scroll to Bottom on Load and New Messages
**Date:** 2026-04-24 | **Category:** UX / Technical | **Status:** Final
**Decision:** `ChatThread.jsx` rewrote scroll-to-bottom logic. Initial load uses `setTimeout(0)` to fire after React paint. Each unloaded image gets `img.addEventListener('load', scrollToBottom)` to handle photos loading after initial scroll. Fixes Main Chat not scrolling to bottom when images are present. Back button moved to right side of thread header, labeled `← Back`.

---

## DEC-130 — Foursomes R2/R4: Holes Thru Indicator on Match Cards
**Date:** 2026-04-24 | **Category:** UX | **Status:** Final
**Decision:** Added "Thru X" indicator to R2/R4 team match cards in `Foursomes.jsx`. Single indicator per match shown in the VS column below match status. Computed as max thru across all four players in the match. Matches the existing R1 "Thru X" pattern.

---

## DEC-131 — Three-State Round: scoringOpen Field Added to Round Doc
**Date:** 2026-04-27 | **Category:** Architecture | **Status:** Final
**Decision:** Added `scoringOpen: boolean` field to round docs alongside existing `isLocked`. Three states: Scheduled (`!isLocked && !scoringOpen`), Scoring (`!isLocked && scoringOpen`), Locked (`isLocked`). Both fields always written together via `updateDoc(docRefs.round(...), { isLocked: newLocked, scoringOpen: newScoringOpen })`. All existing `isLocked` gates (winnings, skins, prize calc, Complete Round) unchanged.

---

## DEC-132 — Three-State Round: Admin Tile Cycle with Sequential Enforcement
**Date:** 2026-04-27 | **Category:** UX / Architecture | **Status:** Final
**Decision:** Admin round tile tapping cycles Scheduled → Scoring → Locked → Scheduled. Sequential rules: advancing to Scoring requires previous round ≥ Scoring or locked (R1 always ok); advancing to Locked requires previous round Locked; unlocking (Locked → Scheduled) requires next round not Locked. Tile shows calendar icon (blue-gray) for Scheduled, open lock (green) for Scoring, closed lock (red) for Locked. Hint text: "Tap to cycle: Scheduled → Scoring → Locked".

---

## DEC-133 — Dashboard Player View: Three States Tied to Round State
**Date:** 2026-04-27 | **Category:** UX | **Status:** Final
**Decision:** Player Dashboard tee time section has three distinct states:
1. **Enter Scores** — shown when `!isLocked && scoringOpen`. Shows tee time card + "⛳ Enter Round X Scores" button.
2. **Scores Submitted** — shown when `myScoresSubmitted && !isLocked`. `myScoresSubmitted = myFoursome?.completedAt != null && !!(scoresByRound[activeRoundNum]?.some(s => s.playerId === user?.uid))`. Shows "✅ Scores Submitted / Waiting for round to be finalized…".
3. **Round Complete** — shown when `myFoursomeComplete = activeRound?.isLocked ?? false`. Shows "✅ Round X Complete" card with gross/to-par, group members, "View Groups →".
Dual condition on `myScoresSubmitted` prevents showing submitted state after scores are wiped.

---

## DEC-134 — Tournament Complete: Manual Admin Action, Decoupled from R4 Lock
**Date:** 2026-04-27 | **Category:** Architecture | **Status:** Final
**Decision:** R4 locking no longer auto-advances `outing.status` to `complete`. The status sync `useEffect` in Dashboard now derives max status as `round4` (not `complete`) when all 4 rounds are locked. A "🏆 Lock Tournament" button appears in Admin Controls when all 4 rounds are locked and status is not yet `complete`. Writes `status = 'complete'` on tap. Same button becomes "🔓 Reopen Tournament" when `status === 'complete'`, writing `status = 'round4'`. This gives admins control over when the TournamentSummaryCard appears.

---

## DEC-135 — Tournament Status Card: Dynamic Title from Round State
**Date:** 2026-04-27 | **Category:** UX | **Status:** Final
**Decision:** STATUS_CONFIG no longer provides static title/color/icon for round1–round4. These are computed dynamically as `displayTitle`, `displayColor`, `displayIcon` based on active round state: "Round X Scheduled" + gray when `!scoringOpen && !isLocked`; "Round X UNDERWAY" + green when `scoringOpen`; "Round X Complete" + gold when `isLocked`. setup/complete statuses still use static STATUS_CONFIG values.


---

## DEC-136 — Post-Freeze Exception: Dining/Announcement Modals → Inline Forms
**Date:** 2026-06-13 | **Category:** Architecture / UX | **Status:** Final
**Decision:** Code freeze (per SessionStarter, citing "DEC-122") was exempted for a critical pre-outing UX fix. `AnnouncementModal` and `DiningEventModal` in `InfoScreen.jsx` (both rendered via the shared `.modal-overlay`/`.modal` floating overlay pattern) were converted to inline forms — `AnnouncementForm` and `DiningEventForm` — shown in-page when `showForm` is true, hiding the list, matching the existing `NotesTab`/`RulesTab` pattern. No modal overlay remains in `InfoScreen.jsx`.
**Rationale:** On iOS PWA standalone, the floating `.modal-overlay`/`.modal` pattern made the Save button on the "Add Dining Event" form unreachable — the modal's bottom (including Save) was hidden behind the fixed bottom nav, and neither the modal nor the background scrolled to reveal it. Multiple CSS-only fixes were attempted and reverted (see "Attempts" below) before converting to the inline-form pattern resolved it completely with no side effects.
**Attempts considered/reverted:**
1. Added `body.modal-open { position: fixed; overflow: hidden; width: 100% }` + `useModalScrollLock` hook — caused `.bottom-nav` (also `position: fixed`) to become contained by `body`'s new fixed positioning, so the nav scrolled/translated with the modal instead of staying pinned to the viewport. Reverted.
2. Tried `.modal-overlay { overflow-y: auto }` + `-webkit-overflow-scrolling: touch` on both `.modal-overlay` and `.modal` — did not resolve gesture capture on iOS.
3. Diagnostic test (`max-height: 70dvh; background: red` on `.modal`) confirmed `.modal` itself sizes correctly relative to viewport and sits above the bottom nav by z-index — the issue was purely overlay-pattern scroll behavior on iOS standalone, not z-index or `dvh` miscalculation.
**Resolution:** `.modal`/`.modal-overlay` CSS fully reverted to pre-session state. `useModalScrollLock.js` deleted. `InfoScreen.jsx`: `AnnouncementModal`/`DiningEventModal` replaced with inline `AnnouncementForm`/`DiningEventForm` cards, toggled via `showForm` state — same pattern as `NotesTab`.
**Alternatives considered:** JS-measured viewport height via `window.visualViewport` instead of `dvh` — not needed once inline-form pattern was adopted.
**Note:** SessionStarter.md's code-freeze line cites "DEC-122", but DEC-122 in this log is "r3Leaderboard Derived from scrambleResults" — these do not match. This appears to be a pre-existing citation error in SessionStarter.md, unrelated to this session's work. Flagged for correction, not altered here.
**Status:** Final — pattern (inline forms over floating modals for full-page forms on mobile/iOS PWA) should be the default going forward; documented in BestMethods/TechnicalArchitecture.

---

## DEC-137 — Player Profile Card on Info Screen
**Date:** 2026-06-16 | **Category:** Feature / UX | **Status:** Final
**Decision:** Added a "My Profile" card to the Info screen home grid (full width,
above the INFO section). Tapping opens a `ProfileEditModal` bottom sheet with
selfie upload + phone number + favorite casino game + firstYear (admin only).
Photo uploads to Firebase Storage `profiles/{uid}/selfie.jpg`; URL saved to
`users/{uid}.profilePhotoUrl`. `yearsPlayed` is always computed from `firstYear`
as `currentYear - firstYear + 1` — never stored. `firstYear` set by admins only;
set for all 32 players via one-time bulk script. Modal uses flex-column layout
with scrollable fields area and sticky Save/Cancel footer to prevent buttons
being obscured by content or bottom nav. Modal z-index: 2000. Footer
padding-bottom: `calc(env(safe-area-inset-bottom) + 72px)`.

---

## DEC-138 — Players Directory Screen
**Date:** 2026-06-16 | **Category:** Feature / UX | **Status:** Final
**Decision:** Added `PlayersScreen.jsx` at `/players` route — a 2-column tile
grid showing all 32 enrolled players. Navigated to from an "All Players" card
in the Info screen INFO section. Data loaded lazily via `useAllPlayers.js` hook
(`onSnapshot` on `queries.enrolledPlayers(outingId)` — only fires when screen
mounts). Default sort: years descending; admins float to top naturally because
they have the most years. Sort controls: Years / First name / Last name. Each
tile shows: photo or initials avatar, name, years badge, admin badge, phone
(tap-to-call), favorite casino game. Initials color is deterministic from
displayName char-code hash — consistent per player across sessions. Admin badge
driven by `ADMIN_DISPLAY_NAMES` constant — never hardcoded.

---

## DEC-139 — Payments Collection Read Rule Opened to All Signed-In Players
**Date:** 2026-06-21 | **Category:** Security / Architecture | **Status:** Final
**Decision:** `payments` Firestore read rule (both the `outings/{outingId}/payments`
subcollection block and the top-level fallback block) changed from
`allow read: if isAdmin() || request.auth.uid == resource.data.playerId` to
`allow read: if isSignedIn()`. Write access unchanged — remains
`isAdmin() || request.auth.uid == request.resource.data.playerId`. Players can
now read every player's payment doc, including `entryFeePaid`, `skinsOptIns`,
and `paidOutFlags` — not just their own.
**Rationale:** Post-freeze exception found after R1. `useSkins.js` builds live
skins standings via `queries.paymentsForOuting` — a broad query across all
payment docs. Firestore rejects any query that could return documents outside
what the per-document read rule permits, so for non-admins the entire query
silently failed (no error handler), leaving `payments` permanently `[]` on the
client. This made `skinsByRound[round]` permanently `null` for every player, so
`SkinsTable.jsx` showed its empty state ("Skins results will appear as scores
are entered") regardless of actual score data — admins, who pass `isAdmin()`,
were the only ones who ever saw the live board. Tom confirmed it's acceptable
for all 32 players (a private friend group) to see each other's payment/paid-out
status in exchange for the simplest fix.
**Alternatives considered:** Restructuring `skinsOptIns` into a separate,
less-sensitive doc/collection readable by all while keeping `entryFeePaid`/
`paidOutFlags` admin-only — rejected as unnecessary scope/risk this close to R2
given the privacy tradeoff was acceptable.
**Note:** Root cause confirmed via code trace (`MoneyScreen.jsx`,
`AdminPayments.jsx`, `useSkins.js`, `SkinsTable.jsx` — all use identical logic,
no `isAdmin` branching anywhere in the UI layer) plus `firestore.rules` — this
was a rules-only bug, not a UI gating difference. Winnings tab (`WinningsTab` in
`MoneyScreen.jsx`) was unaffected — it only reads `rounds`/`matches`/
`scrambleResults`/`outing`, none of which are payments-gated, and is correctly
gated by round lock per DEC-127 identically for all users.

---

## DEC-140 — Tournament Results PDF Export
**Date:** 2026-06-24 | **Category:** Feature / Architecture | **Status:** Final
**Decision:** Built a client-rendered tournament results page at `/results-pdf`
(admin, with print button) and `/history/:year` (player-accessible, no print button).
Single component `TournamentResultsPDF.jsx` uses `useLocation` to detect route and
conditionally show the print bar. Admin Panel home screen shows "📄 Export Results PDF"
button when `outing.status === 'complete'`. User taps browser print → Save as PDF.
No Cloud Function, no Firebase Storage upload, no file management needed.
**Sections:** Tournament header + courses + purse; age group winners; notable
achievements (low round, most birdies, eagles); quick-nav anchor links; final
standings (all 32 players); top 5 overall/prize/skins; per-round leaderboards
R1–R4 with match results; skins hole-by-hole per opted-in round.
**Skins table:** Transposed — players as rows, holes 1–18 as columns, skins won
count in final column. Skins sections use `@page landscape` print CSS. Only
opted-in players shown (correct by design — skins is an optional buy-in).
**Print CSS:** `.bottom-nav`, `.sync-bar`, `.app__main` hidden via `@media print`.
Quick-nav links hidden in print via `.no-print` class.
**Data sources:** `useLeaderboard`, `useSkins`, `useOutingSetup`, round docs
(onSnapshot), score docs (onSnapshot). All prize calculations reuse existing
`calculatePrizePools`, `calculateScramblePrizes`, `prizePerMatchWinner` from
`prizes.js`. Top 5 prize/skins logic ported directly from `TournamentSummaryCard`
in `Dashboard.jsx`.
**Alternatives considered:** Cloud Function PDF generation (puppeteer/pdfkit) —
rejected as unnecessary complexity; browser print-to-PDF is indistinguishable for
sharing purposes.

---

## DEC-141 — History Entry Point in Info Screen
**Date:** 2026-06-24 | **Category:** Feature / UX | **Status:** Final
**Decision:** Added a 🏆 History card to the Info screen home grid. Tapping
navigates to `/history/2026` using the same navigate-not-setActiveSection pattern
as the existing Players card. Route `/history/:year` added to `App.jsx` without
AdminGuard — any logged-in player can view results. When Phase 8A builds the
`historicalTournaments` Firestore collection, the `/history` route (no year param)
will render a year-picker list; tapping a year navigates to `/history/:year`.
Currently `/history/2026` renders live 2026 data via `TournamentResultsPDF`;
future years will render from historical import data once Phase 8A is complete.

---

## DEC-142 — Historical Data Import Schema
**Date:** 2026-06-24 | **Category:** Architecture | **Status:** Final
**Decision:** Historical data (2011–2025) stored in dedicated Firestore collections
separate from live tournament collections to avoid collisions with Phase 8A
multi-tournament architecture. Collections: `historicalTournaments/{year}`,
`historicalResults/{year}_{roundNumber}_{playerName}`,
`historicalStandings/{year}_{playerName}`, `historicalPlayers/{playerName}`,
`courses/{courseName}_{tees}`.
**Source:** `UP_Golf_Historical_Import_v3.xlsx` — 6-sheet Excel (Tournaments,
Players, Courses, Rounds, Results, Standings). Tom audited and confirmed final.
**Import method:** One-time Node.js script reads Excel via xlsx library, writes
to Firestore. Script to be built in Phase 8A.
**Key schema decisions:**
- `payout` in Results = per-round only (never tournament total)
- `finalRank` lives in Standings sheet only, not Results
- `yearsPlayed` stored as comma-separated year list; `yearsCount` computed from length
- `tees` per player per round (required for future handicap calc)
- `gameFormat` controlled vocabulary: `stroke_individual`, `stroke_stableford`,
  `better_ball_gross`, `better_ball_stableford`, `scramble_2team`…`scramble_8team`,
  `better_ball_match`
- `handicapUsed`: Y / N / Unknown per round
- Courses sheet: one row per course+tee combo with slope + rating (pre-filled
  from web research for all 5 UP Golf courses; Royal St Patrick's flagged for
  scorecard verification)

---

## DEC-143 — CC Prompts Always Delivered as Downloadable MD Files
**Date:** 2026-06-24 | **Category:** Process | **Status:** Final
**Decision:** All CC prompts are delivered as downloadable `.md` files via
`create_file` tool, never as inline chat text. Naming convention:
`CC_[ShortDescription].md`. All files presented together via `present_files`
at end of the code-delivery step. This was added to MASTER_CLAUDE_PROTOCOL.md
section 4. Applies to all projects, all sessions.
**Rationale:** MD files can be opened and copied in full without scrolling or
truncation risk; files persist in session for reference if Claude Code needs
to be re-run; naming makes it clear which file each prompt targets.

---

## DEC-144 — Session Docs Fetched via GitHub URLs, Never Uploaded Manually
**Date:** 2026-06-24 | **Category:** Process / Architecture | **Status:** Final
**Decision:** All project docs (SessionStarter, DecisionLog, IssuesTracker,
ProjectRoadmap, TechnicalArchitecture, BestMethods, TimeLog) moved to the
public `dataforge-standards` GitHub repo under `up-golf-pwa/` subfolder.
MASTER_CLAUDE_PROTOCOL.md lives at the repo root. Claude fetches all docs
via `web_fetch` at session start using raw GitHub URLs configured in the
Claude Project Instructions field. No manual file uploads ever needed.
Session end: Claude generates CC prompt MD files → Tom pastes into Claude Code
→ commits and pushes `dataforge-standards` → live for next session automatically.
**Rationale:** Eliminates manual upload friction, ensures Claude always has
latest committed version, single source of truth in GitHub.
**Repo:** github.com/Whit19/dataforge-standards (public)

---

## Session — June 29, 2026

**DEC-145 — History collections use prefixed top-level names**
`history_tournaments`, `history_rounds`, `history_results`, `history_standings` — prefixed to avoid collision with live data collections. All four are top-level (not subcollections), consistent with existing app architecture.

---

**DEC-146 — Historical-only players stored by name only (no uid)**
Players without app credentials (uid: null) in history_ collections. Active players (32) linked by UID at import time. If a historical player returns, linking done manually by admin.

---

**DEC-147 — Off-season nav: 3-tab mode**
When `outing.status === 'complete'`, bottom nav shows Home | History | Info (+ Admin for admins). Full 5-tab nav shown during active tournament. Driven by `useOutingStatus` hook reading live outing doc.

---

**DEC-148 — History doc IDs: year-based for 2011–2025, outingId-based going forward**
Imported historical docs use year as ID ("2025"). Future archived tournaments use outingId ("outing_2026"). Migration to unified outingId scheme deferred to Phase 8B.

---

**DEC-149 — Archive to history is admin-triggered (manual)**
No Cloud Function. AdminArchiveTournament component reads live Firestore data and writes to history_ collections via client SDK. Admin confirms before running. Safe to re-run (set() overwrites).

---

**DEC-150 — TournamentResultsPDF stays hardcoded to ACTIVE_OUTING_ID for now**
Phase 8B will add URL param support. "Full Results" button in history only shows for current outing (matched by outingId field on tournament doc).

---

**DEC-151 — Tabler Icons added via CDN**
`@tabler/icons-webfont` loaded from jsdelivr CDN in index.html. Matches Club Golf icon system. Used for history nav icon (ti ti-history) and header.

---

**DEC-152 — App renamed to Tourney Golf**
Display name only — Firebase project ID (up-golf-pwa), repo name, and local folder unchanged. index.html title and apple-mobile-web-app-title updated. Paired app Club Golf covers single-round play; Tourney Golf covers multi-day tournament outings.

---

**DEC-153 — Player career stats exclude scramble rounds from scoring averages**
Scramble round scores are team scores, not individual. Excluded from avg score calculation and shown as '—' in standings tables. Detection: gameFormat.startsWith('scramble').

---

**DEC-154 — BestMethods.md raw URL uses refs/heads/main path**
Correct URL: `https://raw.githubusercontent.com/Whit19/dataforge-standards/refs/heads/main/up-golf-pwa/BestMethods.md`
Standard `/main/` path returns 404. Update Claude Project Instructions to use refs/heads/main for all 7 doc URLs.

---

## Session — June 30, 2026

**DEC-155 — AdminArchiveTournament.jsx reads only fields that actually exist on live docs**
`payments.prizeWon`/`payments.skinsWon` (nonexistent fields) replaced with
`payments.winningsEarned` (real, now-persisted field). `outing.location`/
`outing.startDate`/`outing.endDate` (nonexistent — outings doc has no such
fields) replaced with values derived from `rounds[].courseName` (joined,
deduplicated) and `rounds[].date` (min/max). `rounds.gameFormat` source read
corrected to `rounds.format` — the `history_rounds` destination field name
stays `gameFormat` to match the 2011–2025 imported schema (DEC-142); only the
live-doc source field name was wrong.

---

**DEC-156 — functions/prizes.js: loud failure replaces silent empty-batch writes**
`calculateAndWriteMatchResults` and `calculateAndWriteScrambleResults` now
count actual writes against docs found and throw a descriptive error if data
existed but nothing was updated, instead of committing an empty batch
successfully. The final `prizesCalculatedAt` write (the idempotency guard) is
now also gated on this check — it will not be written for a round that
produced no real results, so a future failure of this kind can retry on the
next lock event rather than being permanently skipped by the guard. Also: the
hardcoded `r2Pool / 8 / 2` match-prize calculation in
`recalculateAllPlayerWinnings` was replaced with a count derived from actual
`matchesSnap` data per round (R2/R4 separately), matching the equivalent live
calculation already used in `AdminPayments.jsx` (`prizePerMatchWinner(pool,
matchList.length || 8)`), consistent with BestMethods.md's
format-driven-not-hardcoded principle. `prizePerPlayer` is now also written
directly onto each `matches` doc alongside `winner`, so a doc reader never
needs to re-derive the dollar amount.
**Rationale:** Discovered via Tom's outing_2026 archive cleanup request —
`matches.winner` and `matches.team1CombinedScore`/`team2CombinedScore` were
`null` on real, locked-round match docs despite `onRoundLocked`'s logs
showing successful completion and `prizesCalculatedAt` already set on every
round (every subsequent trigger logged "already calculated — skipping
duplicate trigger"). The original cause of why the very first invocation's
batch wrote nothing was never conclusively root-caused (see ISS-149) — this
fix prevents recurrence and makes any future instance immediately visible in
logs, rather than resolving the historical mystery.

---

**DEC-157 — Scramble place-assignment tie-detection uses the full tiebreaker chain**
`calculateAndWriteScrambleResults`'s place-assignment step previously compared
only `totalGross` to decide whether two consecutive sorted teams tied for the
same place — but the sort immediately above it already breaks ties using
`backNine` then `lastThree` (per DEC-052/ISS-052). This meant any time
multiple teams shared the same `totalGross` (already correctly separated by
the sort via tiebreakers), the place-assignment step collapsed them into one
shared place anyway, paying the wrong prize tier. Fixed: tie detection now
requires `totalGross`, `backNine`, AND `lastThree` to all match before two
teams are treated as genuinely tied.
**Confirmed via real data:** outing_2026 R3 had three teams (T5, T2, T8) all
at `totalGross: 61`; before the fix, all three were assigned `place: 2`. T5's
real place (2nd, by `backNine: 30` vs the others' `32`) is independently
confirmed by the live `AdminPayments.jsx` screen, which has separately and
correctly computed prize tiers all along via different logic
(`calculateScramblePrizes`) — this is what made the bug detectable: the
backfilled Firestore data disagreed with the already-trusted live UI.
**Note:** An earlier fix attempt this session targeted a suspected
self-referencing-array `ReferenceError` in the same code block. That issue
may or may not be real depending on JS engine specifics, but was confirmed
NOT to be the actual cause of the wrong-prize-tier bug — the real cause is
the tie-detection condition described above. Both issues were fixed together
in the same code block since they were adjacent.

---

**DEC-158 — Single source of truth for match/skins/scramble winnings: Firestore, not live recompute**
`AdminPayments.jsx`'s `CollectPaySection` no longer recomputes match winners
(`resolvedMatches`), prize-per-player (`calculatePrizePools`/
`prizePerMatchWinner`), or scramble winnings (`calculateScramblePrizes`)
client-side on every render. It now reads `matches.winner`/
`matches.prizePerPlayer` and `payments.scrambleWinnings` directly — the
persisted values written by `functions/prizes.js` (DEC-156/157). `useSkins.js`
already correctly preferred a persisted `skinsResults` doc over its
client-side fallback computation (`if (storedResults[roundNumber])` check) —
only its header comments were updated to stop describing the Cloud Function
as disabled/hypothetical, since it has been live since 2026-03-30 (ISS-036).
**Rationale:** The live-recompute pattern in `AdminPayments.jsx` was originally
built as a deliberate fallback in March 2026 while `onRoundLocked` was
disabled (ISS-036), explicitly documented as such in `useSkins.js`'s header
comment at the time. Once ISS-036 was resolved and the function deployed
2026-03-30, nobody reverted the fallback — it silently became the permanent
path, and the two systems (live recompute, persisted Firestore writes) drifted
completely apart with zero observable symptom, since the UI always showed
correct numbers regardless of what was actually stored. This was only
discovered because Tom asked to archive `outing_2026` and the archive script
read the (wrong) persisted values instead of the (correct) UI values.
**Verification:** A one-time snapshot script captured the live UI's
per-player winnings as ground truth BEFORE any code changed (manually
spot-verified against real paid-out amounts in the Payments screen — Jeff
Searing, Danny Sanicki, etc.). After the fix, a backfill script manually
invoked the corrected calculation logic against the already-locked
`outing_2026` (Firestore triggers don't re-fire on already-true `isLocked`)
and diffed every player's result against the snapshot. Final diff: 32/32
players matched exactly once the scramble place-detection bug (DEC-157) was
also fixed — all "mismatches" before that fix were the snapshot's own R3
figures being unset (since `scrambleResults` had never successfully been
written for `outing_2026`), not errors in the new persisted data.
**Status:** Final — this is now the standing pattern. Any future prize/winnings
display work should read persisted Firestore values, never recompute from
raw scores/matches client-side.

---

**DEC-159 — TournamentResultsPDF.jsx: historical mode reads history_ collections via :year param**
New hook `useHistoricalResults.js` reads `history_tournaments`/
`history_rounds`/`history_results`/`history_standings` and reshapes the data
into the same row shapes `useLeaderboard`'s `cumulativeLeaderboard` already
produces, so existing JSX (Final Standings table, etc.) renders either source
without duplicated rendering logic. `TournamentResultsPDF.jsx` now derives
`isHistoricalMode` from the `:year` URL param — `true` whenever the requested
year is not the currently active outing's year. In historical mode, live
hooks (`useLeaderboard`, `useSkins`, `useOutingSetup`) are still called
(React hooks rules require unconditional calls) but their output is ignored
in favor of `useHistoricalResults`'s output. `totalPurse` reads
`history_tournaments.totalPurse` directly in historical mode instead of
recomputing from `outing.buyIn * playerCount`. `top5Prize` in historical mode
is derived from `history_standings` sorted by `totalPayout` descending,
labeled generically (`sources: ['final payout']`) since per-category
attribution isn't available at that granularity in `history_results`.
**Rationale:** This was a known, previously-logged gap (DEC-150) —
`TournamentResultsPDF.jsx` always read `ACTIVE_OUTING_ID` regardless of the
`:year` param, meaning `/history/2025` would have silently shown whatever
tournament was currently active rather than 2025's actual archived results,
once a tournament other than 2026 became active. Addressed proactively this
session while the archive/payout work was already in progress, ahead of
Phase 8B.
**Known gaps (not addressed this session, see ISS-148):** Per-round
leaderboard detail tables (R1–R4 with match results) and hole-by-hole skins
grids render empty in historical mode — no historical equivalent of these
data shapes was built. `ageGroup` is not present in `history_standings`'s
current schema, so the Grp column shows `—` for archived years.
**Doc ID assumption:** Historical mode currently assumes `:year` is a plain
year string matching `history_tournaments` doc IDs (correct for 2011–2025
imports per DEC-148). A future outingId-based archived tournament (post Phase
8B doc ID migration) will need a follow-up — out of scope for this fix.
**Tested:** `/history/2025` confirmed rendering correctly end-to-end
(standings, payout breakdown, course/format/dates). `/results-pdf` and
`/history/2026` confirmed unchanged (both still resolve to live-data mode,
since 2026 is the current active outing).

---

**DEC-160 — Full payout breakdown by category added to TournamentResultsPDF**
New "Payout Breakdown" table added to `TournamentResultsPDF.jsx`, placed
between Final Standings and Top 5 Winners. Ranked highest-paid to $0,
including $0 players (not filtered out, consistent with the ranking
convention established earlier this session). Live mode shows six columns
(R1 Skins, R2 Skins, R2 Match, R3 Scramble, R4 Skins, R4 Match) built from
`matches.winner`/`prizePerPlayer`, `skinsByRound[rn].playerSummary`, and
`payments.scrambleWinnings` — all persisted values per DEC-158, not
recomputed. Historical mode shows four columns (R1–R4) since
`history_results.payout` is only available at per-round granularity, not
broken out by category within a round.
**Rationale:** Tom requested visibility into how each player's total
winnings break down by source, beyond the single aggregate "Won" figure
already shown in Final Standings.
