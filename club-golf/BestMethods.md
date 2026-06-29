# BestMethods.md — Club Golf Platform
**Hard-won lessons from UP Golf and Club Golf v1. Apply before writing any new code.**

---

## How to Use This Document
Before starting any new feature or phase, read the relevant section here first.
These are not suggestions — they are proven fixes to real bugs that cost hours to track down.

---

## 1. PWA & Build

| Rule | Detail |
|------|--------|
| Always `--legacy-peer-deps` | vite-plugin-pwa has peer dep conflicts — never plain `npm install` |
| `injectManifest` + `rollupFormat: 'iife'` | Only SW strategy that works correctly with Firebase + iOS |
| Keep project outside OneDrive | `node_modules` + OneDrive = corruption and build errors |
| PowerShell: no `&&` chaining | Run commands sequentially. `npm run build` then `firebase deploy` |
| `index.html` must be `no-cache` | Add to `firebase.json` — browsers will serve stale entry point forever otherwise |
| SW files must be `no-cache` | Same reason — stale SW = invisible updates |
| Immutable cache for `*.js` / `*.css` | Hashed filenames are safe to cache forever |
| VAPID key: copy/paste only | Never retype — one wrong character = push silently fails forever |
| PWA icons: white background | 180px (Apple touch), 192px, 512px. Transparent backgrounds render badly on iOS |
| `user-scalable=no` is an accessibility violation | Use `maximum-scale=5.0` instead |
| `<main id="root">` in `index.html` | Satisfies ARIA landmark requirement — Lighthouse Accessibility |
| `window.confirm()` silently blocked in PWA standalone | Always use inline confirmation UI (a panel with Yes/No buttons) |

---

## 2. Firebase & Firestore

| Rule | Detail |
|------|--------|
| All collection/doc refs through `firestorePaths.js` | Never inline Firestore refs in components — one path change breaks everything otherwise |
| All constants in `constants.js` | Never hardcode collection names, status strings, or format labels in JSX |
| `enableIndexedDbPersistence()` is deprecated | Use `persistentLocalCache` API instead |
| Offline cold-start: `setLoading(false)` must NOT be inside `await loadProfile()` | If Firestore is offline, `loadProfile` never resolves — app spins forever |
| `onSnapshot` for real-time reads; `setDoc/merge` for writes | Standard pattern across both apps — don't mix `getDocs` into real-time flows |
| Firestore composite index | Create in Firebase console when query combines `where` + `orderBy` on different fields |
| Member doc IDs are NOT Firebase Auth UIDs | ID is auto-generated. Firebase Auth UID stored as `uid` field. Never assume they match |
| `includeMetadataChanges: true` on critical snapshots | Prevents stale cached status on app reopen |
| Firestore path cleanup is non-negotiable | Every file that reads Firestore must go through `firestorePaths.js` — do a full audit before each phase ships |
| Every new Cloud Function needs "Allow public access" in Cloud Run | Not set automatically — must be done manually in Cloud Run console after first deploy |
| `functions/.env` is never committed | Gmail credentials, API keys — always in `.env`, always in `.gitignore` |

---

## 3. Authentication

| Rule | Detail |
|------|--------|
| `displayName` fallback is `''` not `user?.email` | Using email causes displayName flash on load — shows email address briefly |
| `lastLoginAt: serverTimestamp()` on every login | Written in `useAuth.jsx` — enables "X players activated" admin tracking |
| Invite flow: admin pre-creates, member claims | Never allow open self-registration — use tokenized email invite links |
| Token is single-use, 7-day expiry | Longer = security risk, shorter = frustrating |
| `deleteMember` Cloud Function removes BOTH Firestore doc AND Auth account | Deleting only the Firestore doc leaves orphaned Auth accounts that can't be recreated |
| After `claimInvite`: auto sign-in and redirect to `/games` | Don't make the user log in manually after claiming their invite |

---

## 4. Score Entry & Game State

| Rule | Detail |
|------|--------|
| Three-state model: Scheduled → Scoring → Locked | Always write `isLocked` and `scoringOpen` TOGETHER — never one without the other |
| Sequential enforcement on state transitions | Scoring requires previous round ≥ Scoring; Locked requires previous round Locked |
| `Enter Scores` button requires BOTH `!isLocked AND scoringOpen` | Not just `!isLocked` — this was a real bug in UP Golf |
| `myFoursomeComplete` driven by `isLocked` only | NOT `completedAt` — `completedAt` is for "submitted" state, lock is for "complete" |
| `myScoresSubmitted` requires BOTH `completedAt != null` AND score doc exists | Dual condition prevents showing submitted state after score wipe |
| Game Complete is manual admin action | Never auto-advance to complete on final lock — gives admin control over when results appear |
| Winnings/results gated on round lock | Never show live winnings mid-round — only after organizer locks |
| `FORMAT_LABELS` from constants, never hardcoded in STATUS_CONFIG | Prevents format text drift when formats change |
| Score entry UI: one thumb operation | All tap targets reachable with thumb, minimum 44px touch targets |
| `position: sticky` unreliable in flex overflow containers | Use `position: fixed` for headers in scroll containers |
| Fixed header height: `ResizeObserver` + `el.offsetHeight` | NOT `contentRect.height` — causes layout bugs. Add `paddingTop: headerHeight + 10` to scrollable container |
| Resume hole on reload | Initialize `currentHole` to `null`, detect first unplayed hole on load — don't default to hole 1 |
| Default score to hole par | Never hardcode default gross score — always use `hole.par` fallback |

---

## 5. Scoring Engine Rules

| Rule | Detail |
|------|--------|
| Net score = gross - strokesReceived | `strokesReceived = 1 if hole.strokeIndex <= player.playingHandicap` |
| `holes[n].scores[playerId]` is `{ gross, net }` object | NOT a plain number — always destructure |
| Stats update: always pass `{ ...game, totals: newTotals }` | Stale `game.totals` snapshot causes hole 18 points to be missing from stats |
| `(raw + bonus) × multiplier = total` | Bonus points are additive BEFORE the multiplier — not after |
| Birdie point: gross beats net at same score | Eagle beats birdie; lower score wins; true tie cancels |
| Basket check uses gross-only birdie pass | Separate from net birdie calc — prevents basket/casket failing in `net_no_basket` mode |
| Stats are non-blocking | `updateMemberStats` should never block the score save flow |
| Guest players: skip stats | Only write stats for players with a `memberId` |
| `scoringOpen` field: always write with `isLocked` | `updateDoc(roundRef, { isLocked: x, scoringOpen: y })` — atomic write |
| Stats are a FLAT object on member doc | Access as `memberDoc.stats` not `memberDoc.stats.six_point` — no format nesting in v1 |

---

## 6. Leaderboard & Results

| Rule | Detail |
|------|--------|
| Leaderboard sorts by score-to-par, not raw gross | Raw gross is misleading — always show +/- against par played |
| Tiebreaker check uses `.some()` not `.every()` | 3-way ties need ANY shared score to trigger display — `.every()` misses them |
| `scrambleResults` must be declared before `r3Leaderboard` in `useLeaderboard.js` | Wrong order = ReferenceError — JavaScript hoisting does not help here |
| `r3Leaderboard` derived from `scrambleResults` | Not sorted independently — ensures Leaderboard and MoneyScreen always match |
| Default tab to active round: use `hasDefaulted` ref pattern | Fires once only on load — doesn't override user tab changes mid-session |
| Tiebreaker scores shown for ALL tied entries | Not just the winner — show `Back 9 Score: X` / `Last 3 Holes Score: X` for every tied row |
| Match status: always "Up X", never "Down X" | Show the leader's perspective from the trailing team's angle is confusing |
| "Thru X" indicator on live match cards | Compute as max `thru` across all four players in the group |

---

## 7. Multi-Tenant Architecture

| Rule | Detail |
|------|--------|
| All data under `clubs/{clubId}/` — no global collections | Isolation is the entire multi-tenant model |
| `clubId` must travel through invite links explicitly | Don't rely on localStorage for new device first-logins |
| `clubs/{clubId}/profile` is the source of truth for club name/branding | Never hardcode club name in the app — always read from profile doc |
| `clubs/{clubId}/roles/{uid}` for role checks | `isAdmin()` uses `get()` lookup — keep roles collection separate from member docs |
| Member doc write rules: admin-only | Exception: members can write their own `favorites`, `handicapIndex`, `defaultTees`, `venmoHandle` |
| Document ID prefixes for sub-entities | e.g. `G` prefix for foursomes, `M` for matches — prevents ID collisions when multiple entity types share a collection |

---

## 8. UX Patterns That Worked

| Pattern | Detail |
|---------|--------|
| Game Day Card on home screen | Single card with all game-day info — player shouldn't need to navigate to find their game |
| Three-state Dashboard (Scheduled / Scoring / Locked) | Players always know exactly where they stand in the round lifecycle |
| Orientation-driven scorecard | Landscape = traditional scorecard, Portrait = points view. Zero UI, natural gesture |
| Favorites in player picker | Top section of modal with gold star — saves game creation time for regular groups |
| Collapsible leaderboard card | Collapsed by default in History — keeps primary content (game list) front and center |
| Player history per format type | The stat that matters: "my record at this game" — not aggregate points |
| All/Mine toggle on History | Simple filter that makes personal history immediate |
| Inline confirmation for destructive actions | Panel with "← Go Back / Confirm ✓" — no `window.confirm()` |
| `completedAt` on foursome doc | Lightweight signal that players finished — doesn't require admin lock |
| "Complete Round" button with summary | Shows per-player gross + to-par before confirming — players verify before submitting |
| Single scrolling feed over tabs | GamesPage: no tabs — Active → Upcoming → Last Round in one scroll. Tabs hide information. |
| Stats strip on landing page | Player's own W/L/pts diff always visible — instant context without navigating |
| FAB for primary action | Floating "+ New Game" button bottom-right — thumb-accessible, doesn't consume header space |

---

## 9. Things That Were Harder Than Expected

| Area | Why It Was Hard | Mitigation |
|------|----------------|------------|
| Push notifications | FCM token accumulation, duplicate delivery, iOS foreground suppression, compat SDK conflicts | Defer to later phase. Use native SW `push` event only. Replace token array, never append |
| Chat scroll | Images load after initial render — naive `useEffect` on messages fails | `setTimeout(0)` + `img.addEventListener('load', scrollToBottom)` |
| Offline cold-start | `await loadProfile()` never resolves when Firestore is offline | `setLoading(false)` must be outside the await |
| Firestore path drift | Inline refs accumulate across components and then break silently when paths change | `firestorePaths.js` from day one — enforce strictly |
| Stats on hole 18 | Stale game snapshot passed to stats update — hole 18 points always missing | Always pass `{ ...game, totals: newTotals }` |
| Scorecard in PWA landscape | iOS safe area, fixed positioning, viewport scaling all interact badly | `position: fixed` + all four safe-area insets inline |
| Status drift in admin | `outing.status` and round lock states got out of sync | Status derived from lock chain, never set manually |
| iOS date input in flex container | WebKit native date rendering ignores flex width constraints — overflows container | Add `-webkit-appearance: none; appearance: none` to input AND `min-width: 0` to parent flex div |
| Roster horizontal scroll | Layered overflow containers conflict — inner scroll ignored by outer | Use dedicated `roster-table-scroll` wrapper div with its own `overflow-x: auto`; remove overflow-x from outer container; add `page-container--wide` to remove max-width cap |
| Clipboard API in PWA | `navigator.clipboard` undefined in some browser/PWA contexts (Edge tracking prevention) | Always guard with `navigator.clipboard?.writeText` and fall back to `textarea + execCommand('copy')` |

---

## 10. Document & Session Workflow

| Rule | Detail |
|------|--------|
| Upload all 6 docs at session start | SessionStarter, ProjectRoadmap, DecisionLog, IssuesTracker, TechnicalArchitecture, TimeLog |
| Design/data model decisions in this (Projects) session | Code is delivered as CC prompts for VS Code — don't mix design and implementation |
| Upload relevant files when debugging logic bugs | Don't ask CC to guess at root causes for data-critical fixes |
| Update all docs at session end | SessionStarter is the most critical — it's the only context that carries forward |
| TimeLog: update at session end | Duration + one-sentence summary — invaluable for estimating future phases |
| Next available decision ID | Check DecisionLog before adding — `CGAD-045` is next for Club Golf |
| Next available issue ID | Check IssuesTracker — `CGI-054` / `CGI-B046` / `CGI-D023` are next for Club Golf |

---

*Last updated: May 2026 — synthesized from UP Golf (Phase 7 complete) + Club Golf (Session 20 complete)*
