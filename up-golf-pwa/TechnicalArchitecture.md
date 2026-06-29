# TechnicalArchitecture.md — UP Golf PWA
**Full technical specification: stack, data models, API structure, offline strategy, push notifications, and file structure**

---

## 1. Technology Stack

### Frontend
| Layer | Technology | Notes |
|-------|-----------|-------|
| Language | JavaScript (ES2022+), JSX | React hooks-based; no class components |
| Framework | React 18 | |
| Build Tool | Vite | Use `--legacy-peer-deps` with vite-plugin-pwa |
| CSS | Custom Properties + utility classes | Golf-themed design system |
| PWA | vite-plugin-pwa (Workbox) | `injectManifest` strategy; `rollupFormat: 'iife'`; single SW |
| Offline Storage | Firestore native offline persistence | Firestore handles most cases natively |
| Icons | Custom PNG icons | 180px (Apple touch), 192px, 512px — white background |

### Backend
| Layer | Technology | Notes |
|-------|-----------|-------|
| Platform | Firebase | Firestore, Auth, Storage, Functions, Hosting |
| Database | Firebase Firestore | NoSQL; real-time `onSnapshot`; native offline persistence |
| Auth | Firebase Auth (email/password) | `isAdmin: true` in Firestore user doc; enforced server-side |
| File Storage | Firebase Storage | Photo uploads by outing/day/round |
| Server Functions | Firebase Cloud Functions (Node.js) | Prize calc, push triggers, score locking, bulk import, birdie notifications |
| Push Notifications | Firebase Cloud Messaging (FCM) | iOS 16.4+ required; fully validated on iOS and Android |

### Hosting
| Platform | Notes |
|----------|-------|
| Firebase Hosting | Blaze pay-as-you-go (activated); CDN-backed; HTTPS |
| Domain | GoDaddy CNAME → Firebase Hosting |
| Cache headers | `index.html` → `no-cache`; SW files → `no-cache`; `**/*.@(js|css)` → `immutable` |

### Dev Environment
| Tool | Notes |
|------|-------|
| OS | Windows / PowerShell — no `&&` chaining |
| Node | v24 / npm v11 — always `--legacy-peer-deps` |
| IDE | VS Code |
| Project path | `C:\Dev_Projects\up-golf` — outside OneDrive |
| Firebase SDK | 12.11.0 frontend; firebase-admin for functions |
| firebase-messaging-sw.js | No longer uses Firebase compat SDK — native push event only |

---

## 2. Data Models

### `users`
```js
{
  id: string,               // Firebase Auth UID
  username: string,
  displayName: string,
  email: string,
  isAdmin: boolean,
  ageGroup: 'under50' | 'over50',
  birthYear: number,
  enrolledOutings: string[],
  fcmTokens: string[],      // Single-element array — always replaced, never appended
  lastLoginAt: timestamp,   // Written on every login in useAuth.jsx
  createdAt: timestamp,
}
```

### `rounds` — top-level, ID = `{outingId}_R{roundNumber}`
```js
{
  outingId: string,
  roundNumber: number,
  format: string,           // stroke_play | combined_gross | scramble | best_ball
  courseName: string,
  courseId: string,
  timezone: string,
  date: string,
  teeTimes: string,         // newline or comma separated; split with /[\n,]+/
  teesUpper: string,
  teesLower: string,
  parValues: number[],
  skinsEnabled: boolean,
  skinsBuyIn: number,
  isLocked: boolean,
  prizesCalculatedAt: timestamp | null,
}
```

### `outings` — ID = `{outingId}`
```js
{
  id: string,
  status: string,           // setup|round1|round2|round3|round4|complete
  buyIn: number,
  playerCount: number,
  prizeAllocations: { round2Pct, round3Pct, round4Pct },
}
```

### `foursomes` — ID = `{outingId}_R{roundNumber}_G{groupNumber}`
```js
{
  outingId: string,
  roundNumber: number,
  groupNumber: number,
  teamNumber: number,       // R3 only — matches scrambleTeams teamNumber
  teeTime: string,          // populated by generatePairings for all rounds including R3
  playerIds: string[],      // R2/R4: [team1[0], team1[1], team2[0], team2[1]]
  matchNumber: number,      // R2/R4 only
  completedAt: timestamp | null,  // written when players tap "Complete Round" ✅
  updatedAt: timestamp,
}
```

### `matches` — ID = `{outingId}_R{roundNumber}_M{matchNumber}`
```js
{
  outingId: string,
  roundNumber: number,
  matchNumber: number,
  team1PlayerIds: string[],
  team2PlayerIds: string[],
  team1Seeds: number[],
  team2Seeds: number[],
  teeTime: string,
  winner: null | 'team1' | 'team2' | 'tie',
  prizePerPlayer: number | null,
  updatedAt: timestamp,
}
```

### `scrambleTeams` — ID = `{outingId}_T{teamNumber}`
```js
{
  id: string,
  outingId: string,
  teamNumber: number,
  players: [{ playerId, slot, seed, tees }],
  updatedAt: timestamp,
}
```
Note: No `teeTime` on scrambleTeams — look up from foursomes doc by teamNumber.

### `rules` — ID = `{outingId}`
```js
{
  sections: [
    {
      title: string,
      icon: string,
      rules: [{ heading: string, body: string }]
    }
  ],
  updatedAt: timestamp,
}
```
Firestore security: admin write, auth read required.

---

### History Collections (Phase 8A)

Four read-only collections store archived tournament data. Imported via Node.js
Admin SDK scripts for 2011–2025; future years archived via AdminArchiveTournament.

**history_tournaments**
Doc ID: year string for imports ("2025"), outingId for archived ("outing_2026")
Fields: year, name, startDate, endDate, location, playerCount, totalPurse,
        outingId (archived only), notes, importedAt, archivedAt

**history_rounds**
Doc ID: {tournamentDocId}_R{roundNumber}
Fields: year, outingId, roundNumber, course, gameFormat, handicapUsed,
        holesPlayed, par, notes, importedAt

**history_results**
Doc ID: {tournamentDocId}_R{roundNumber}_{uid|playerName}
Fields: year, outingId, roundNumber, playerName, uid (null for historical-only),
        score, score18, holesPlayed, payout, teamId, skinsWon, notes, importedAt

**history_standings**
Doc ID: {tournamentDocId}_{uid|playerName}
Fields: year, outingId, playerName, uid (null for historical-only),
        finalRank (null if no ranking recorded), totalPayout, importedAt

### Composite Indexes (added Phase 8A)
- history_standings: year ASC + finalRank ASC
- history_standings: uid ASC + year DESC
- history_rounds: year ASC + roundNumber ASC

### New Files (Phase 8A)
- src/hooks/useOutingStatus.js — subscribes to active outing doc, returns {outingStatus, isOffSeason}
- src/screens/HistoryScreen.jsx — /history route; Tournaments + Players tabs
- src/components/AdminArchiveTournament.jsx — admin tool to archive live outing to history_
- functions/fetch-uids.js — one-time utility (delete after use)
- functions/import-history.js — one-time utility (delete after use)

---

## 3. Round Lock → Status Flow (DEC-072)

```
All rounds unlocked         → outing.status = 'round1'
R1 locked                   → outing.status = 'round2'
R1 + R2 locked              → outing.status = 'round3'
R1 + R2 + R3 locked         → outing.status = 'round4'
R1 + R2 + R3 + R4 locked    → outing.status = 'complete'
```

---

## 4. Admin Navigation Pattern (DEC-073)

- Bottom nav always visible for authenticated users including `/admin`
- Admin header: title + subtitle on left, "← Menu" button on right (only when inside a section)

---

## 5. Fixed Header Dynamic Height Pattern (DEC-077)

```js
const headerRef = useRef(null);
const [headerHeight, setHeaderHeight] = useState(FALLBACK_PX);
useEffect(() => {
  const el = headerRef.current;
  if (!el) return;
  const ro = new ResizeObserver(() => setHeaderHeight(el.offsetHeight));
  ro.observe(el);
  return () => ro.disconnect();
}, []);
// Scrollable container:
<div style={{ paddingTop: headerHeight + 10 }}>
```
**Critical:** Use `el.offsetHeight` not `contentRect.height`. Applied to: `Foursomes.jsx`, `LeaderboardScreen.jsx`, `AdminPanel.jsx`

---

## 6. Push Notifications — Fully Validated ✅ (DEC-115, DEC-116, DEC-117)

### Architecture
- Token storage: `users/{uid}.fcmTokens[]` — single-element array; always replaced, never appended
- `broadcastPush` — admin-callable; UI built in AdminPanel home screen
- `notifyFoursomesPosted` / `notifyRoundLocked` — wired into Cloud Functions
- `onScoreBirdie` / `onScrambleBirdie` — Firestore triggers with dedup guard
- Dead token pruning server-side after each multicast

### Single Delivery Path (DEC-115)
- **All notifications handled by SW native `push` event** in `firebase-messaging-sw.js`
- Firebase compat SDK (`importScripts`, `firebase.initializeApp`, `onBackgroundMessage`) removed from SW entirely
- `onMessage` foreground handler removed from `usePushNotifications.js` entirely
- FCM token registration still uses modular SDK in React app — SW needs no Firebase at all

### SW push event handler
```js
self.addEventListener('push', (event) => {
  const payload = event.data?.json() ?? {};
  const { title, body } = payload.notification ?? {};
  const { url } = payload.data ?? {};
  event.waitUntil(
    self.registration.showNotification(title || 'UP Golf', {
      body, icon: '/icons/icon-192.png', badge: '/icons/icon-192.png',
      vibrate: [100, 50, 100], data: { url: url || '/' },
      actions: [{ action: 'open', title: 'Open App' }],
    })
  );
});
```

### Token Management (DEC-116)
- `subscribe()` writes `fcmTokens: [token]` — replaces entire array
- `checkAndRefreshToken()` writes `fcmTokens: [freshToken]` if token changed or array has >1 token
- `unsubscribe()` calls `pushManager.getSubscription().unsubscribe()` first, then wipes `fcmTokens: []`
- Bell badge in Dashboard: `unsubscribe()` → `subscribe()` — always produces new token

### FCM Payload (DEC-117)
- All sends include `data: { url: '/path' }` for SW click navigation
- `fcmOptions.link` kept as fallback
- `birdieNotify.js` sends `data: { url: '/leaderboard' }`

---

## 7. Player Activation Tracking (DEC-080)

- `lastLoginAt: serverTimestamp()` written on every login in `useAuth.jsx`
- AdminPlayers summary banner: "X / Y players activated"
- Per-player badge: green "✓ Xd ago" or gray "never logged in"

---

## 8. scrambleResults Shape (DEC-085)

```js
{
  teamId: string,
  teamNumber: number,
  teamName: string | null,
  totalGross: number,
  backNine: number,
  lastThree: number,
  place: number,
  holes: HoleScore[],
  players: EnrichedPlayer[],
  playerIds: string[],
}
```

---

## 9. Accessibility (DEC-097–099)

- **Viewport:** `maximum-scale=5.0` — pinch zoom allowed; `user-scalable=no` removed
- **Landmark:** `<main id="root">` in `index.html` — satisfies ARIA main landmark
- **Login contrast:** `login-screen__sub` and `login-screen__footer` use `rgba(255,255,255,0.75)` on dark green

### Lighthouse Baseline (2026-04-02)
| Category | Score |
|----------|-------|
| Performance | 85 |
| Accessibility | 95 |
| Best Practices | 100 |
| SEO | 82 |
| PWA | N/A — removed from Lighthouse in Chrome 116+ |

---

## 10. Three-State Round System (DEC-131, DEC-132, DEC-133, DEC-134, DEC-135)

### Round States
| State | `isLocked` | `scoringOpen` | Player sees | Admin tile |
|-------|-----------|---------------|-------------|------------|
| Scheduled | false | false | Tee times, no score button | Gray/blue, calendar icon |
| Scoring | false | true | Tee times + Enter Scores button | Green, open lock |
| Locked | true | false | Round Complete card | Red, closed lock |

### Field
`scoringOpen: boolean` on round doc — always written together with `isLocked`:
```js
updateDoc(docRefs.round(ACTIVE_OUTING_ID, rn), { isLocked: newLocked, scoringOpen: newScoringOpen })
```

### Sequential Enforcement (cycleRoundState)
- Scheduled → Scoring: prev round must be ≥ Scoring or Locked (R1 always ok)
- Scoring → Locked: prev round must be Locked (unchanged)
- Locked → Scheduled: next round must not be Locked (unchanged)

### Dashboard Player States
- `myScoresSubmitted = myFoursome?.completedAt != null && !!(scoresByRound[activeRoundNum]?.some(s => s.playerId === user?.uid))`
- `myFoursomeComplete = activeRound?.isLocked ?? false`
- State 1 (scoringOpen): tee time card + Enter Scores button
- State 2 (myScoresSubmitted && !isLocked): "✅ Scores Submitted / Waiting for round to be finalized…"
- State 3 (isLocked): "✅ Round X Complete" card

### Tournament Complete (DEC-134)
- R4 lock does NOT auto-advance status to `complete`
- Status sync derives max as `round4` when all 4 locked
- Admin "🏆 Lock Tournament" button appears when all 4 locked and status ≠ complete
- Writes `status = 'complete'` → TournamentSummaryCard appears
- "🔓 Reopen Tournament" writes `status = 'round4'`

### Tournament Status Card Title (DEC-135)
Computed as `displayTitle/displayColor/displayIcon` from active round state:
- `!scoringOpen && !isLocked` → "Round X Scheduled", gray
- `scoringOpen` → "Round X UNDERWAY", green
- `isLocked` → "Round X Complete", gold

## 11. Complete Round Flow (DEC-126)

- **Trigger:** All 18 holes entered for all players (individual) or team (scramble), OR round is already locked
- **Button:** "✓ Complete Round" appears in `ScoreEntry.jsx` below sync status row
- **Confirmation:** Panel shows per-player gross + to-par summary; "← Go Back" / "Confirm ✓" buttons
- **Firestore write:** `updateDoc(doc(db, 'foursomes', myFoursome.id), { completedAt: serverTimestamp() })`
- **Navigation:** After confirm, navigates back (-1)
- **Dashboard:** `myFoursomeComplete = myFoursome?.completedAt != null || activeRound?.isLocked`
- **Round Complete card:** Shows ✅ Round X Complete, final gross/to-par (from `scoresByRound`), group members, "View Groups →" link (navigates `/foursomes`)
- **Rainout:** Admin locks early → `isLocked` drives complete state; `completedAt` not required

---

## 12. Winnings Lock Gate (DEC-127)

### Dashboard `myWinnings`
- R2 match winnings: `rounds[2]?.isLocked`
- Scramble winnings: `rounds[3]?.isLocked`
- R4 match winnings: `rounds[4]?.isLocked`
- Skins: per-round `rounds[roundNum]?.isLocked`

### MoneyScreen `WinningsTab`
- `rounds` prop passed from parent (already loaded via `onSnapshot`)
- `roundLocked = rounds?.[activeRound]?.isLocked ?? false`
- If not locked: "🔒 Results will appear once Round X is locked" placeholder
- If locked: normal match/scramble results display

---

## 13. Font Standards (DEC-128)

- **UI font:** `var(--font-ui)` — all screens, leaderboard numbers, score cells
- **Mono font:** `var(--font-mono)` — admin import/setup/player screens for code-like data only
- **Removed:** `font-family: var(--font-mono)` from `.lb-table__seed-num`, `.lb-table__score`, `.lb-table__tpar` in `index.css`
- **Normalised:** Bare `'monospace'` strings in AdminBulkImport, AdminRoundSetup, AdminPlayers → `var(--font-mono)`
- **Removed:** Redundant `fontFamily: 'inherit'` from AdminPayments, AdminPanel, InfoScreen

---

## 14. File & Folder Structure

```
/up-golf/
├── src/
│   ├── screens/
│   │   ├── Dashboard.jsx         // Round Complete card; winnings lock gate; dynamic format label
│   │   ├── AdminPanel.jsx
│   │   ├── LeaderboardScreen.jsx
│   │   ├── Foursomes.jsx         // R2/R4 Thru X indicator on match cards
│   │   ├── MoneyScreen.jsx       // Winnings lock gate; hasDefaultedSkins added
│   │   ├── ScoreEntry.jsx        // Complete Round button + confirmation + completedAt write
│   │   ├── ChatScreen.jsx        // Back button on right side
│   │   └── InfoScreen.jsx
│   ├── components/
│   │   ├── PhotoGallery.jsx
│   │   ├── Leaderboard.jsx       // Tiebreaker .some() fix; R3 cell styling fix
│   │   ├── ChatThread.jsx        // Scroll-to-bottom with image load handling
│   │   ├── AdminFoursomes.jsx
│   │   ├── AdminRoundSetup.jsx
│   │   ├── AdminScoreOverride.jsx
│   │   ├── AdminPayments.jsx
│   │   ├── AdminPlayers.jsx
│   │   ├── AdminBulkImport.jsx
│   │   ├── AdminCourseLibrary.jsx
│   │   └── PushPermissionPrompt.jsx
│   ├── hooks/
│   │   ├── usePushNotifications.js
│   │   ├── useLeaderboard.js     // backNine/lastThree on r1 rows; scrambleResults before r3Leaderboard
│   │   └── useAuth.jsx
│   └── utils/
│       ├── constants.js
│       └── firestorePaths.js
├── public/
│   ├── firebase-messaging-sw.js  // Native push event only; no Firebase compat SDK
│   └── icons/
├── functions/
│   ├── index.js
│   ├── push.js
│   ├── birdieNotify.js
│   ├── generatePairings.js
│   └── prizes.js
├── firebase.json
├── index.html
├── index.css                     // font-mono removed from leaderboard number cells
└── App.jsx
```

---

## 15. Phase 8 Architecture — Multi-Tournament, Net Scoring & Flexible Formats

*(See SessionStarter.md Phase 8 Summary and ProjectRoadmap Phase 8 for full task list)*

> **Status:** Not Started. Planned post-UP-outing-2026.

### 14A — Multi-Tournament: New/Extended Data Models
*(Full models documented in previous TechnicalArchitecture — unchanged)*

### 14B — Course Data API Integration
**Primary option:** GolfCourseAPI.com — free tier, ~30,000 courses.
**Fallback/upgrade:** GolfAPI.io — paid, 42,000+ courses.

### 14C — Net Scoring Engine
**Course Handicap formula (WHS standard):**
```
courseHandicap = Math.round(handicapIndex × (slope / 113) + (courseRating - par))
```

### 14D — Flexible Game Format Engine
| `formatType` | Description | `countingScores` |
|---|---|---|
| `individual_gross` | Standard stroke play, gross | n/a |
| `individual_net` | Gross minus course handicap | n/a |
| `best_ball_net` | Best N net scores per hole | number |
| `variable_best_ball_net` | Best N net, N varies by par | `{ par3, par4, par5 }` |
| `scramble` | Existing scramble engine | n/a |
| `stableford_gross` | Points vs par, gross | n/a |
| `stableford_net` | Points vs par, net | n/a |

### 14E — `useActiveTournament.js` Hook
Replaces the `ACTIVE_OUTING_ID` constant pattern entirely. Reads `config/active` → resolves `tournamentId` → subscribes to `tournaments/{id}`.

---

## 16. Known Pitfalls

| Issue | Note |
|-------|------|
| PowerShell `&&` | Not supported — run commands sequentially |
| OneDrive + node_modules | Keep project outside OneDrive |
| vite-plugin-pwa peer deps | `npm install --legacy-peer-deps` |
| Outing ID | Always `outing_2026` with underscore |
| SW iife format | `injectManifest` must use `rollupFormat: 'iife'` |
| VAPID key | Copy/paste only — never retype |
| fcmTokens field | Always replace with `[token]` — never use `arrayUnion` |
| SW cache headers | `Cache-Control: no-cache` in `firebase.json` for SW files |
| index.html cache | Must be `no-cache` — browsers will serve stale entry point otherwise |
| ResizeObserver for headers | Use `el.offsetHeight` not `contentRect.height` |
| Push dual delivery | Do NOT use Firebase compat SDK in SW — registers its own push listener causing duplicates |
| Push foreground (iOS) | iOS suppresses notifications when app is active foreground — OS behavior, not a bug |
| Push token refresh | `checkAndRefreshToken` on mount — replaces array if token changed or length > 1 |
| Push token accumulation | Always use `fcmTokens: [token]` not `arrayUnion` |
| Push unsubscribe | Must call `pushManager.getSubscription().unsubscribe()` to force fresh token on re-subscribe |
| Push click navigation | SW reads `payload.data.url` — must be set in FCM message `data` field |
| birdieNotify.js independent | Has own sendBirdieNotification — separate from push.js sendMulticast |
| Push success ≠ delivered | FCM returns success even when iOS silently drops notification |
| generatePairings push timing | 2s delay after batch.commit() |
| onRoundLocked notification | Must be called BEFORE prizesCalculatedAt guard |
| Birdie dedup | Cloud Functions v2 fires at-least-once — always use event ID dedup |
| `window.confirm()` | Silently blocked in PWA standalone — use inline confirmation |
| `position: sticky` | Unreliable in flex overflow containers — use `position: fixed` |
| Round lock status | Always derived from lock chain — never set `outing.status` manually |
| scoringOpen field | Always write both isLocked and scoringOpen together — never write one without the other |
| Tournament complete | status=complete is manual only — R4 lock does NOT auto-set complete |
| myFoursomeComplete | Driven by isLocked only — not completedAt |
| myScoresSubmitted | Requires both completedAt != null AND score doc exists for player |
| Enter Scores button | Requires both !isLocked AND scoringOpen — not just !isLocked |
| teeTimes storage | Split with `/[\n,]+/` |
| scrambleResults | Must expose `playerIds` and `holes` for prizes.js |
| R3 tee times | Stored on foursomes docs (not scrambleTeams) |
| Rules collection | Requires Firestore security rule: admin write, auth read |
| age group normalisation | `normaliseAge()` returns `'U50'`/`'50+'` |
| Inline Firestore refs | ALL collection/doc refs go through firestorePaths.js |
| Lighthouse PWA score | Removed in Chrome 116+ — not applicable |
| user-scalable | Must NOT be set to `no` — accessibility violation |
| Offline cold-start | `setLoading(false)` must NOT be inside `await loadProfile()` |
| Offline sync status | Use `useSyncStatus().isOnline` for UI |
| Push prompt flash | `isSupportChecked` must gate `isInitialising` |
| Admin score entry | `isAdmin` bypass in ScoreEntry uses `selectedFoursomeId` |
| Player usernames | Full FirstnameLastname format — app appends `@upgolf.local` |
| Foursome doc ID | Must use `_G{n}` suffix — e.g. `outing_2026_R2_G1` |
| Match doc ID | Must use `_M{n}` suffix — e.g. `outing_2026_R2_M1` |
| AdminFoursomes save | Must use `getDocs` not `onSnapshot` |
| AdminFoursomes R2/R4 | Save must write BOTH foursomes + matches docs |
| Skins opt-in cutoff | `isClosed = round?.isLocked` only |
| payments collection reads | Open to all signed-in users (`isSignedIn()`), not just admin/own-doc — changed DEC-139 so useSkins.js's broad paymentsForOuting query works for non-admins. Write stays admin-or-own-doc. Any future broad query against a collection must check whether the read rule is per-document-owner-scoped, which silently breaks list/collection-group queries for non-owners. |
| displayName in useAuth | Fallback is `''` not `user?.email` |
| scrambleResults before r3Leaderboard | In `useLeaderboard.js`, `scrambleResults` useMemo MUST be declared before `r3Leaderboard` |
| Scramble team names | Set by admin only in AdminFoursomes |
| sortWithTiebreaker rows | R1/R2/R4 rows have `backNine` and `lastThree` fields; R1 rows attached directly |
| Default tab on load | Use `hasDefaulted` ref pattern — fires once only |
| Tiebreaker check | Use `.some()` not `.every()` — 3-way ties need any shared score to trigger |
| Dashboard format text | Derived from `FORMAT_LABELS[activeRound?.format]` — not hardcoded in STATUS_CONFIG |
| Complete Round button | Writes `completedAt` to foursome doc — use `doc(db, 'foursomes', myFoursome.id)` directly |
| Dashboard Round Complete | `myFoursomeComplete = myFoursome?.completedAt != null \|\| activeRound?.isLocked` |
| Winnings lock gate | Both Dashboard myWinnings and MoneyScreen WinningsTab gate on `rounds[n]?.isLocked` |
| Groups nav route | `/foursomes` — not `/groups` |
| Chat scroll | `ChatThread.jsx` uses setTimeout(0) + image load handlers — not a simple useEffect on messages |
| Leaderboard font | `.lb-table__seed-num`, `.lb-table__score`, `.lb-table__tpar` use var(--font-ui) not var(--font-mono) |
| Floating modal overlay (iOS PWA) | `.modal-overlay`/`.modal` pattern can become unscrollable/unreachable on iOS standalone (Save button hidden behind bottom nav, no vertical scroll). Body-level `position: fixed` scroll-lock breaks `.bottom-nav` (also fixed) by changing its containing block. Prefer inline forms (toggle `showForm`, hide list) over floating modals for full-page forms — see NotesTab, AnnouncementForm, DiningEventForm (DEC-136) |
