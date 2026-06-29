# TechnicalArchitecture.md — Club Golf Platform
**Full technical spec: data models, file structure, Firestore paths, auth, calc engines, design system**
*Updated May 2026 — Session 24 complete*

---

## Stack

| Layer | Technology | Notes |
|-------|-----------|-------|
| Frontend | React 18 + Vite | Hooks only, no class components |
| Styling | CSS Variables — global stylesheet (index.css) | No CSS modules |
| Routing | React Router v6 | |
| PWA | vite-plugin-pwa (injectManifest strategy) | `rollupFormat: 'iife'` — required |
| Backend / DB | Firebase Firestore | Real-time `onSnapshot`; native offline persistence |
| Auth | Firebase Auth (email/password) | |
| Hosting | Firebase Hosting | Blaze plan; CDN-backed |
| Functions | Firebase Cloud Functions (Node.js 24, v2 API) | |
| Email | Nodemailer + Gmail App Password | tjunker9@gmail.com |
| Package mgmt | npm — always `--legacy-peer-deps` | |

**Local project path:** `C:\Dev_Projects\club-golf`
**Firebase project ID:** `club-golf-9f946`
**Production URL:** `https://club-golf.app` (custom domain live 5/31/26 — GoDaddy DNS + Firebase Hosting SSL)
**PowerShell:** No `&&` chaining — run commands sequentially
**uuid package:** installed — used for gameId, readOnlyToken, invite tokens

---

## Critical Build Rules

| Rule | Why |
|------|-----|
| `npm install --legacy-peer-deps` always | vite-plugin-pwa peer dep conflict |
| `injectManifest` + `rollupFormat: 'iife'` | Only SW strategy that works with Firebase + iOS |
| `index.html` → `Cache-Control: no-cache` in firebase.json | Browsers serve stale entry point forever otherwise |
| SW files → `no-cache` | Same reason |
| Project must be outside OneDrive | node_modules + OneDrive = corruption |
| `window.confirm()` silently blocked in PWA standalone | Use inline confirmation UI always |
| `user-scalable=no` is accessibility violation | Use `maximum-scale=5.0` |
| All Firestore refs through `firestorePaths.js` | One path change breaks everything otherwise |
| All constants in `constants.js` | Never inline collection names or status strings |

---

## PWA Update Banner Pattern
```js
// In App.jsx
import { useRegisterSW } from 'virtual:pwa-register/react';

const { needRefresh: [needRefresh], updateServiceWorker } = useRegisterSW();

// Render when needRefresh is true:
// Fixed banner at top: "A new version is available" + "Update Now" button
// Button calls: updateServiceWorker(true)
```
- Uses `vite-plugin-pwa/react` — no SW file changes needed
- Only appears when app is open during a deploy and new SW is waiting
- Members who are fully closed get the update automatically on next open

---

## iOS Safe Area Rule
**Every page root div must use `className="page-container"`**
- Applies `padding-top: max(1rem, env(safe-area-inset-top))`
- Never add additional safe-area padding inside page-specific headers
- **Exceptions:** JoinPage (`.join-page`), ScorecardView (`position: fixed` full-viewport)
- **Wide pages** (e.g. RosterPage): use `className="page-container page-container--wide"` to remove 480px max-width cap

---

## iOS Date Input in Flex Container
```css
/* On the input itself */
input[type="date"] {
  -webkit-appearance: none;
  appearance: none;
  min-width: 0;
}
/* On the parent flex div */
.date-wrapper {
  min-width: 0;
}
```

---

## Fixed Header Height Pattern
```js
const headerRef = useRef(null);
const [headerHeight, setHeaderHeight] = useState(56);
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
**Critical:** Use `el.offsetHeight` NOT `contentRect.height`.

---

## Clipboard API Fallback
```js
async function copyToClipboard(text) {
  try {
    if (navigator.clipboard?.writeText) {
      await navigator.clipboard.writeText(text);
    } else {
      const el = document.createElement('textarea');
      el.value = text;
      el.style.position = 'fixed';
      el.style.opacity = '0';
      document.body.appendChild(el);
      el.select();
      document.execCommand('copy');
      document.body.removeChild(el);
    }
  } catch (_) {
    // Silent fail
  }
}
```

---

## Roster Horizontal Scroll Pattern
```jsx
<div className="page-container page-container--wide">
  <div className="admin-layout">
    <div className="admin-layout__list">        {/* overflow-y: auto only */}
      <div className="roster-table-scroll">     {/* overflow-x: auto */}
        <table className="data-table">...</table>
      </div>
    </div>
  </div>
</div>
```

---

## HI Age Pattern
```js
function hiAgeDays(updatedAt) {
  if (!updatedAt) return Infinity;
  return Math.floor((Date.now() - updatedAt.toMillis()) / 86400000);
}
// Color coding: <15d = var(--color-primary), 15–30d = #E6A817, 30+d or null = var(--color-danger)
```
- Always guard for null before calling `toMillis()`
- `handicapUpdatedAt` is written on: ProfilePage `saveHI`, `useRoster` `addMember`/`updateMember`, NewGamePage auto-fill slot 0
- Backfilled from `createdAt` for bulk-imported members via `backfillHandicapUpdatedAt.cjs` (run 5/29/26)

---

## Cloud Functions

| Function | Trigger | Auth Required | Purpose |
|----------|---------|---------------|---------|
| `sendInvite` | onCall | Yes | Generates invite token, emails member |
| `claimInvite` | onCall | No (public) | Validates token, creates Auth account, links UID |
| `deleteMember` | onCall | Yes | Deletes Auth account + Firestore member doc |
| `deactivateMember` | onCall | Yes | Disables Firebase Auth account; sets inviteStatus:'skip' on member doc |
| `reactivateMember` | onCall | Yes | Re-enables Firebase Auth account; sets inviteStatus:'accepted' on member doc |

**After every new function deploy:** Set "Allow public access" manually in Cloud Run console

---

## Multi-Tenant Architecture

All data scoped under `clubs/{clubId}`. No global collections (v1).

```
clubs/
  {clubId}/
    profile
    members/
    courses/
    games/
    roles/
```

---

## Firestore Data Model

### clubs/{clubId}/profile
```js
{
  name: string,
  logoUrl: string | null,
  timezone: string,
  createdAt: timestamp,
  subscriptionTier: "free" | "pro",
  subscriptionStatus: "active" | "inactive",
  ownerUid: string,
  primaryCourseId: string | null,
  settings: {
    allowMemberGameCreation: boolean,
    maxRosterSize: number,
  }
}
```

### clubs/{clubId}/roles/{uid}
```js
{ role: "admin" | "member" }
```

### clubs/{clubId}/members/{memberId}

**IMPORTANT:** Member doc IDs are auto-generated Firestore IDs, NOT Firebase Auth UIDs.

```js
{
  name: string,
  email: string | null,
  handicapIndex: number | null,   // null if not set; plus handicaps stored as negative (e.g. -3.5 for +3.5 HI)
  handicapUpdatedAt: timestamp | null,  // written on every HI save; null = never set
  handicapIndex: number | null,
  defaultTees: string,
  role: "admin" | "organizer" | "member",
  uid: string | null,
  isGuest: false,
  createdAt: timestamp,
  updatedAt: timestamp,
  lastLoginAt: timestamp | null,

  inviteToken: string | null,
  inviteStatus: "pending" | "accepted" | null | "skip",
  inviteExpiresAt: timestamp | null,
  inviteSentAt: timestamp | null,
  accountCreatedAt: timestamp | null,

  favorites: string[],
  tnmlTeam: string | null,
  venmoHandle: string | null,

  // NESTED by year — NOT a flat object (CGAD-046)
  stats: {
    all: {
      gamesPlayed: number,
      wins: number,
      losses: number,
      ties: number,
      pointsFor: number,
      pointsAgainst: number,
      birdies: number,
      proxWins: number,
      baskets: number,
    },
    '2026': { /* same 9 fields */ },
  }
}
```

**Critical:** `stats` is NESTED. Always access as `memberDoc.stats.all` or `memberDoc.stats[year]`. Never `memberDoc.stats.gamesPlayed` directly.

### clubs/{clubId}/courses/{courseId}
```js
{
  name: string,
  createdAt: timestamp,
  updatedAt: timestamp,
  teeSets: [{
    name: string,
    slope: number,
    rating: number,
    par: number,
    holes: [{ holeNumber: number, par: number, strokeIndex: number }]
  }]
}
```

### clubs/{clubId}/games/{gameId}
```js
{
  createdAt: timestamp,
  createdBy: string,
  date: string,               // "YYYY-MM-DD"
  courseId: string,
  status: "scheduled" | "active" | "complete",
  savedToHistory: boolean,
  gameType: 'six_point',
  players: { [playerId]: { ... } },
  playerIds: string[],
  holes: { [1-18]: { ... } },
  totals: { A: number, B: number },
  readOnlyToken: string,
  winner: string | null,
}
```

---

## Game Status Constants

```js
GAME_STATUS = {
  SCHEDULED: 'scheduled',
  ACTIVE:    'active',
  COMPLETE:  'complete',
}
```

---

## Game Lifecycle

```
SCHEDULED → ACTIVE → COMPLETE
```

---

## GamesPage Layout (CGAD-043)

Single scrolling feed — no tabs:
1. **Stats strip** — logged-in player's W/L/Pts Diff/Games from `stats.all`
2. **Active section** — live GameCards
3. **Upcoming section** — scheduled game cards
4. **Last Game** — most recent completed GameCard
5. **Empty state** — shown only if all sections empty

---

## LeaderboardPage Layout (CGAD-047)

Route: `/leaderboard` — dedicated page, protected.
- Year picker: auto-detected from member stat keys; defaults to most recent year
- 4 stat tabs: Points Diff / Wins / Birdies / Prox
- Ranked list; logged-in user highlighted; rank 1 gets gold

---

## BottomTabBar (CGAD-047)

5 tabs for ALL users: **Games / History / Leaderboard / Profile / Info**
Admin Panel accessed via button in ProfilePage.

---

## Scheduling Rules (CGAD-037)

- **Date > today:** Review step shows "Save Game" — always SCHEDULED
- **Date = today:** Review step shows "Start Now" (ACTIVE) or "Save for Later" (SCHEDULED)

---

## Handicap Calculation

```
Course Handicap = round(HandicapIndex × (Slope / 113) + (CourseRating - Par))
Playing Handicap (differential) = CourseHandicap - min(CourseHandicap in group)
Net Score = Gross Score - strokesReceived(hole)
strokesReceived = 1 if hole.strokeIndex <= player.playingHandicap else 0
```

**Plus handicaps:** Stored as negative numbers (e.g. -3.5 for +3.5 HI). Valid range: -10 to 54.

---

## Scoring Engine Critical Rules

| Rule | Source |
|------|--------|
| `holes[n].scores[playerId]` is `{ gross, net }` — never a plain number | Club Golf v1 |
| Always pass `{ ...game, totals: newTotals }` to stats update | Hole 18 bug |
| `(raw + bonus) × multiplier = total` — bonus before multiplier | CGAD-017 |
| Stats write to BOTH `stats.all` and `stats[year]` | CGAD-046 |
| Stats read: `stats.all` for lifetime, `stats[year]` for per-year | CGAD-046 |
| `holeSequence` computed as `Array.from({ length: numHoles }, (_, i) => ((startingHole - 1 + i) % 18) + 1)` | CGAD-045 |

---

## Auth Model

| Role | Firebase Auth | Capabilities |
|------|--------------|-------------|
| Admin | Yes | Full access; Admin Panel via ProfilePage button |
| Organizer | Yes | Create/manage games, add foursomes, lock rounds |
| Member | Yes (via invite) | Create games, enter scores, start rounds they're in |
| Guest player | No | Participate by name only |
| Viewer | No | Read-only link access |

---

## Member Deactivation / Reactivation

```
Deactivate: admin.auth().updateUser(uid, { disabled: true }) + inviteStatus: 'skip'
Reactivate: admin.auth().updateUser(uid, { disabled: false }) + inviteStatus: 'accepted'
```

---

## Firestore Security Rules

```js
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function isAdmin(clubId) {
      return get(/databases/$(database)/documents/clubs/$(clubId)/roles/$(request.auth.uid)).data.role == 'admin';
    }
    match /clubs/{clubId}/profile {
      allow read: if request.auth != null;
      allow write: if request.auth != null && isAdmin(clubId);
    }
    match /clubs/{clubId}/games/{gameId} {
      allow read: if true;
      allow write: if request.auth != null;
    }
    match /clubs/{clubId}/roles/{uid} {
      allow read, write: if request.auth != null && isAdmin(clubId);
    }
    match /clubs/{clubId}/members/{memberId} {
      allow read: if request.auth != null;
      allow write: if request.auth != null && isAdmin(clubId);
    }
    match /clubs/{clubId}/courses/{courseId} {
      allow read: if request.auth != null;
      allow write: if request.auth != null && isAdmin(clubId);
    }
    match /{document=**} {
      allow read, write: if request.auth != null;
    }
  }
}
```

---

## Firestore Indexes Required

| Collection | Fields | Type | Purpose |
|------------|--------|------|---------|
| games | status ASC, createdAt DESC | Composite | activeGames query |
| games | status ASC, date ASC | Composite | scheduledGames query |

---

## Design System

### Brand / UI Colors
| Variable | Value | Usage |
|----------|-------|-------|
| `--color-bg` | #0F1F16 | Page background |
| `--color-surface` | #1e3d28 | Card / panel backgrounds |
| `--color-surface-2` | #2a4a32 | Elevated surfaces |
| `--color-border` | #3a5a42 | Borders |
| `--color-primary` | #4caf7d | Primary action buttons |
| `--color-primary-dark` | #3a9668 | Button hover |
| `--color-text` | #E8F0EB | Primary text |
| `--color-text-muted` | #9ABBA2 | Secondary text |
| `--color-danger` | #e05252 | Destructive actions |
| `--color-gold` | #C8A96E | CRACK indicator, favorites star |
| `--color-basket` | #FFD600 | BASKET indicator |
| `--color-recrack` | #E53935 | RE-CRACK indicator |

### HI Age Colors (inline style, not CSS variables)
| Age | Color |
|-----|-------|
| < 15 days | `var(--color-primary)` — green |
| 15–30 days | `#E6A817` — amber |
| 30+ days or null | `var(--color-danger)` — red |

### CSS Conventions
- BEM double-dash modifiers: `btn--primary`, `btn--ghost`, `btn--full`, `btn--sm`, `btn--danger`
- Page prefixes: `ng-` (NewGamePage), `rp-` (ResultsPage), `hist-` (HistoryPage), `gc-` (GameCard), `sc-` (ScorecardView), `prof-` (ProfilePage), `sgc-` (scheduled game card), `lb-` (LeaderboardPage)
- No CSS modules — global `index.css` only
- No hardcoded hex in JSX — always CSS variables (exception: HI age amber `#E6A817` has no variable)

---

## Routing Structure

```
/                           → redirect to /games
/login                      → LoginPage (public)
/join                       → JoinPage (public — invite claim)
/games                      → GamesPage (protected)
/games/new                  → NewGamePage (protected)
/games/:gameId              → ScoreEntryPage (protected)
/games/:gameId/results      → ResultsPage (protected)
/history                    → HistoryPage (protected)
/leaderboard                → LeaderboardPage (protected)
/profile                    → ProfilePage (protected)
/info                       → InfoPage (protected)
/admin                      → AdminPage (admin only)
/admin/roster               → RosterPage (admin only)
/admin/course               → CoursePage (admin only)
/admin/club                 → ClubProfilePage (admin only)
/viewer                     → ViewerGamesPage (public)
/view/:gameId               → ViewerPage (public)
```

---

## File / Folder Structure

```
club-golf/
├── public/
│   ├── firebase-messaging-sw.js
│   ├── ClubGolf_UserGuide.html
│   ├── robots.txt
│   ├── sounds/
│   │   └── crack.mp3
│   └── icons/
│       ├── icon-180.png
│       ├── icon-192.png
│       └── icon-512.png
├── functions/
│   ├── index.js
│   ├── package.json
│   └── .env                        (never committed)
├── scripts/
│   ├── addFavoritesToMembers.cjs
│   ├── importMembers.cjs
│   ├── migrateStats.cjs
│   ├── updateMemberEmails.cjs      ← 62 emails corrected 5/29/26
│   ├── setDefaultTees.cjs          ← 159 members set to Blue 5/29/26
│   ├── backfillHandicapUpdatedAt.cjs ← 188 members backfilled 5/29/26
│   └── exportMembers.cjs           ← active + pending member CSV export
├── src/
│   ├── components/
│   │   ├── game/
│   │   │   ├── CrackSlider.jsx
│   │   │   └── ScorecardView.jsx
│   │   └── shared/
│   │       ├── ProtectedRoute.jsx
│   │       ├── GameCard.jsx
│   │       ├── BottomTabBar.jsx
│   │       └── InstallPrompt.jsx
│   ├── hooks/
│   │   ├── useAuth.jsx
│   │   ├── useClubProfile.js
│   │   ├── useRoster.js            ← addMember/updateMember write handicapUpdatedAt
│   │   ├── useCourse.js
│   │   ├── useGame.js
│   │   └── useInstallPrompt.js
│   ├── utils/
│   │   ├── constants.js
│   │   ├── firestorePaths.js
│   │   ├── handicapCalc.js
│   │   ├── sixPointEngine.js
│   │   ├── crackEngine.js
│   │   └── statsUtils.js
│   ├── pages/
│   │   ├── LoginPage.jsx
│   │   ├── JoinPage.jsx
│   │   ├── GamesPage.jsx
│   │   ├── NewGamePage.jsx         ← HI Age badge; auto-fill slot 0 writes handicapUpdatedAt
│   │   ├── ScoreEntryPage.jsx
│   │   ├── ResultsPage.jsx
│   │   ├── HistoryPage.jsx
│   │   ├── LeaderboardPage.jsx
│   │   ├── AdminPage.jsx
│   │   ├── ClubProfilePage.jsx
│   │   ├── ViewerPage.jsx
│   │   ├── ViewerGamesPage.jsx
│   │   ├── RosterPage.jsx          ← HI Updated column + HI Age sort
│   │   ├── CoursePage.jsx
│   │   ├── ProfilePage.jsx         ← saveHI writes handicapUpdatedAt; fixed plus HI validation
│   │   └── InfoPage.jsx
│   ├── firebase.js
│   ├── App.jsx
│   └── main.jsx
├── index.css
├── firebase.json
├── firestore.rules
├── .firebaserc
├── vite.config.js
└── package.json
```

---

## Known Pitfalls

| Pitfall | Note |
|---------|------|
| PowerShell `&&` | Not supported — run sequentially |
| OneDrive + node_modules | Keep project outside OneDrive |
| vite-plugin-pwa peer deps | `npm install --legacy-peer-deps` always |
| SW iife format | `injectManifest` must use `rollupFormat: 'iife'` |
| `window.confirm()` | Silently blocked in PWA standalone — inline UI always |
| `position: sticky` | Unreliable in flex overflow — use `position: fixed` |
| ResizeObserver | Use `el.offsetHeight` not `contentRect.height` |
| Offline cold-start | `setLoading(false)` must NOT be inside `await loadProfile()` |
| Stats on hole 18 | Always pass `{ ...game, totals: newTotals }` |
| All Firestore refs | Must go through `firestorePaths.js` |
| Cloud Function public access | Must be set manually in Cloud Run after each new function deploy |
| Stats shape — NESTED not flat | `stats.all.gamesPlayed` NOT `stats.gamesPlayed` |
| Deactivated member uid | Keep uid on member doc through deactivation |
| handicapUpdatedAt — guard null | Always check before calling `.toMillis()` |
| ProfilePage saveHI — plus handicaps | Valid range is -10 to 54, not 0 to 54 |
| useRoster addMember — null HI | Store null when blank, never default to 0 (CGAD-041) |
| NewGamePage auto-fill slot 0 | Must include handicapUpdatedAt in the setSlots patch |

---

*Last updated: Session 24 complete — May 2026*
*Next: Begin sending member invites, Phase 3 TNML planning*
