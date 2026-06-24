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
**Phase 7 (Polish, Testing & Deployment) is COMPLETE ✅**
**Player Profile + Players Directory feature added post-freeze (DEC-136, DEC-137) ✅**
**Outing in progress — Round 1 played 2026-06-20, went smoothly. Rounds 2-4 still ahead.**
**Code is frozen — no further changes until the full outing (all 4 rounds) is complete, beyond narrowly-scoped, logged freeze exceptions (see DEC-136, DEC-139).**

---

## What Was Completed in Phase 7

- Push notifications fully working end-to-end on iOS and Android ✅
- Three-state round system (Scheduled → Scoring → Locked) ✅
- Dashboard player view states (Enter Scores / Scores Submitted / Round Complete) ✅
- Tournament Complete manual admin lock button ✅
- Complete Round confirmation flow with per-player summary ✅
- Winnings and skins gated behind round lock status ✅
- Skins opt-in cutoff fixed (ISS-117) ✅
- In-app User Guide and App Setup PDF guides added to Info screen ✅
- All 32 player credentials distributed ✅
- Admin walkthrough of round controls and Tournament Lock completed ✅
- Dry run with 8 players passed ✅

## What Was Added Post-Freeze (2026-06-16)

- Player Profile card on Info screen — selfie upload, phone, casino game, years played ✅
- Players Directory screen (/players) — 2-col tile grid, sort by years/first/last name ✅
- Storage rules updated to allow profile photo uploads (profiles/{uid}/) ✅
- firstYear field set on all 32 player docs via set-first-years.js bulk script ✅
- ProfileEditModal sticky footer fix — buttons always visible above bottom nav ✅

## What Was Fixed Post-Freeze (2026-06-21)

- Players could not see live Skins standings on Money screen — only admins
  could; players saw a permanent empty state regardless of scores entered.
  Root cause: `firestore.rules` `payments` read rule was scoped to admin/own-doc
  only, which silently broke the broad query `useSkins.js` needs to build the
  skins roster for non-admins. Fixed by opening `payments` read to all
  signed-in users; write access unchanged (ISS-140, DEC-139) ✅

---

## Next Session Priority

Get through Rounds 2-4 of the outing. Phase 8 planning and architecture does
NOT start until the full outing is complete (Tournament Lock pressed, all 4
rounds locked) — see DEC-134. Code stays frozen apart from narrowly-scoped,
logged freeze exceptions for issues that materially affect gameplay/fairness
mid-outing, same bar as DEC-136 and DEC-139.

---

## Phase 8 Summary (Post-UP-Outing — fully scoped, DEC-101–105)

- **8A:** Multi-tournament — `config/active` Firestore doc + `useActiveTournament.js` 
  hook replaces `ACTIVE_OUTING_ID`; `tournaments` + `enrollments` collections
- **8B:** Course data — GolfCourseAPI.com (free, ~30k courses) or GolfAPI.io (paid, 
  42k+); fetched once at admin setup, stored in Firestore `courses` collection; 
  extended schema adds per-hole `strokeIndex`
- **8C:** Net scoring — `handicapIndex` on `users` doc (admin-entered); WHS Course 
  Handicap formula; `useNetScoring.js` hook; per-hole stroke allocation via `strokeIndex`
- **8D:** Flexible formats — data-driven `roundFormats[]` on tournament doc; types: 
  `individual_gross/net`, `best_ball_net`, `variable_best_ball_net` (e.g. 1-2-3, 3-2-3), 
  `scramble`, `stableford_gross/net`; skins in gross or net mode
- **8E:** Admin UX — handicap entry in AdminPlayers; course handicap display in Round 
  Setup; stroke indicator dots on scorecard

---

## Key Architecture Notes

### Firestore Doc ID Conventions (CRITICAL)
All doc IDs must match exactly between `firestorePaths.js` and `generatePairings.js`:
- Foursomes: `{outingId}_R{roundNumber}_G{groupNumber}` — e.g. `outing_2026_R2_G1`
- Matches: `{outingId}_R{roundNumber}_M{matchNumber}` — e.g. `outing_2026_R2_M1`
- Scores: `{outingId}_R{roundNumber}_{playerId}`
- Scramble teams: `{outingId}_T{teamNumber}` — e.g. `outing_2026_T1`
- Rounds: `{outingId}_R{roundNumber}` — e.g. `outing_2026_R1`

### Player Profile + Players Directory (DEC-136, DEC-137)
- **My Profile card** on Info screen home grid — full width above INFO section
- Tapping opens `ProfileEditModal` (bottom sheet) with selfie upload + 3 fields
- Photo uploads to Firebase Storage: `profiles/{uid}/selfie.jpg`
- URL saved to `users/{uid}.profilePhotoUrl` via updateDoc
- Fields on users doc: `profilePhotoUrl`, `phoneNumber`, `casinoGame`, `firstYear`
- `yearsPlayed` always computed as `currentYear - firstYear + 1` — never stored
- `firstYear` editable by admins only in the modal; set for all 32 players via
  bulk script `set-first-years.js` (now deleted — do not recreate)
- **Players Directory** at `/players` route — `PlayersScreen.jsx`
- Hook: `useAllPlayers.js` — lazy `onSnapshot` on `queries.enrolledPlayers(outingId)`
- Default sort: years descending (admins float top naturally via most years)
- Admin badge driven by `ADMIN_DISPLAY_NAMES` constant — never hardcoded
- Initials color: deterministic hash from displayName — consistent per player
- Phone: `<a href="tel:...">` tap-to-call
- Storage rule: `profiles/{uid}/{filename}` — auth read, uid-match write, 5MB, image only
- **Modal z-index: 2000** — above bottom nav (nav z-index is lower)
- **Modal footer padding-bottom: `calc(env(safe-area-inset-bottom) + 72px)`** —
  clears bottom nav height on all iPhone models

### Three-State Round System (DEC-131, DEC-132, DEC-133)
- `scoringOpen: boolean` added to round doc alongside `isLocked`
- Three states: Scheduled (`!isLocked && !scoringOpen`), Scoring (`!isLocked && 
  scoringOpen`), Locked (`isLocked`)
- Admin tile cycles Scheduled → Scoring → Locked on tap; both fields always written 
  together
- Sequential enforcement: advancing to Scoring requires prev round ≥ Scoring; advancing 
  to Locked requires prev round Locked; unlocking requires next round not Locked
- Enter Scores button: `!activeRound.isLocked && activeRound.scoringOpen`
- Player Dashboard states: Enter Scores → Scores Submitted → Round Complete
  - `myScoresSubmitted = myFoursome?.completedAt != null && 
    !!(scoresByRound[activeRoundNum]?.some(s => s.playerId === user?.uid))`
  - `myFoursomeComplete = activeRound?.isLocked ?? false`

### Tournament Complete — Manual Admin Action (DEC-134)
- R4 locking no longer auto-advances `outing.status` to `complete`
- "🏆 Lock Tournament" button appears when all 4 rounds are locked
- Writes `status = 'complete'` → triggers TournamentSummaryCard
- Toggles to "🔓 Reopen Tournament" when already complete

### PDF Guides (added Phase 7)
- Stored in Firebase Storage: `Guides/UP_Golf_App_Installation.pdf` and 
  `Guides/UP_Golf_App_User_Guide.pdf`
- URLs hardcoded as `GUIDE_URLS` constant in `InfoScreen.jsx`
- Displayed as two cards under a "GUIDES" section label on the Info home grid
- Tap opens PDF in device viewer via `window.open(url, '_blank')`
- To update: overwrite the file in Firebase Storage (same path = same URL, no code 
  change needed)

### Push Notification Architecture (fully validated ✅)
- FCM tokens in `users/{uid}.fcmTokens` (array, replaced not appended)
- All notifications handled by SW native `push` event
- No Firebase compat SDK in SW
- Token refresh: `checkAndRefreshToken` runs silently on every mount

### Player Credentials
- Username: full first + last name, mixed case (e.g. `TimFischer`)
- Password: firstname + `26!`, lowercase (e.g. `tim26!`) — admins use `Sweetgrass`
- App auto-appends `@upgolf.local` — players never type the email suffix

### Cache Headers (firebase.json)
- `index.html` → `no-cache`
- SW files → `no-cache`
- `**/*.@(js|css)` → `max-age=31536000, immutable`

### Outing ID
Always `outing_2026` with underscore. Never hyphen.

### PowerShell
No `&&` chaining. Always `npm install --legacy-peer-deps`. Always `npm run build` 
before `firebase deploy --only hosting`. Functions require separate 
`firebase deploy --only functions`.

### VAPID Key (copy/paste only — never retype)
BLcR9lmWMj4kPIpMoo-jT88mu9ZoBO6dIGeGcjmhiNXhLhc0f8Oat8HaLGaxc1x03_p8Ji8EhJFb_arYA-RyN-A

### Auth Pattern
```js
const { user, isAdmin } = useAuth()
import { useAuth } from '../hooks/useAuth.jsx'  // .jsx required
```

### Laptop Dev Setup
- `.env` must be copied manually from desktop — it is gitignored
- No `functions/.env` needed for hosting-only deploys
- Always run `npm install --legacy-peer-deps` on first use of a new machine

---

## Phase Overview

| Phase | Status |
|-------|--------|
| Phase 0–7 | ✅ Complete |
| Phase 8 | 🔲 Not Started (post-outing) |

---

## Open Issues Summary
- **ISS-010** Emoji picker (deferred — post Phase 8)

---

## Session Workflow
1. Load this SessionStarter + upload relevant source files at session start
2. Always upload `constants.js` and `firestorePaths.js` when working on hooks/utilities
3. **NEVER read keys/tokens/IDs from screenshots — always require copy/paste**
4. **Always wait for all necessary files/info before writing code**
5. Work through issues/features
6. At session end: update all 5 docs and provide as downloads
