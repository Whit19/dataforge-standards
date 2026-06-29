# BestMethods.md — UP Golf PWA
**Hard-won lessons from building this app. Read before writing any code.**
Last Updated: 2026-06-29

---

## 1. Firebase / Firestore

### Silent permission failures are the #1 hidden bug class
Firestore collection queries silently return `[]` when the security rule blocks
them — they do not throw an error. A per-document-owner read rule (e.g.
`allow read: if request.auth.uid == resource.data.playerId`) will silently
reject any broad query (`getDocs`, `onSnapshot` on a collection) even if the
user owns many of those docs. Always check rules when a query appears to return
empty data that should exist. (ISS-140, DEC-139)

### Check collection scope before writing any broad query
Before writing a hook that queries an entire collection, verify that the
Firestore security rule permits a list read, not just a per-document read. A
rule that allows individual reads does NOT allow collection-level queries.
This catches a whole class of silent data failures before they reach production.

### All Firestore refs go through `firestorePaths.js`
Never inline a collection name, document path, or query in a component or hook.
Always use `firestorePaths.js` and `constants.js`. Magic strings in JSX = bugs
that are invisible until the wrong doc ID is generated in production.

### Doc ID suffixes are mandatory and exact
- Foursomes: `{outingId}_R{roundNumber}_G{groupNumber}` — the `_G` prefix is required
- Matches: `{outingId}_R{roundNumber}_M{matchNumber}` — the `_M` prefix is required
- Omitting the prefix creates orphan documents that coexist with the correct ones, 
  causing data to appear missing. (ISS-114, ISS-115)

### Always write paired fields together
`isLocked` and `scoringOpen` must always be written in a single `updateDoc` call.
Never write one without the other — stale state combinations cause player-facing 
UI to show the wrong round state. Same principle applies to any pair of fields 
whose meaning is defined by their combination.

### `arrayUnion` accumulates — it never prunes
Using `arrayUnion` for FCM tokens caused stale device tokens to pile up
indefinitely. Always replace arrays entirely: `fcmTokens: [token]`.
Apply this thinking to any array field updated frequently — if old entries become
invalid, append semantics will silently accumulate garbage. (DEC-116)

### `persistentLocalCache` is the correct offline API
`enableIndexedDbPersistence()` is deprecated. Use `persistentLocalCache` API.
Calling deprecated APIs produces warnings and may break in future SDK versions.

### `setLoading(false)` must not be gated on async Firestore calls
If `setLoading(false)` is inside an `await loadProfile()` call and Firestore is
offline, the app spins forever. Always unblock the loading state before or
independent of any awaited Firestore operation. (ISS-095, DEC-107)

### Admin SDK reads only — never commit service account keys
`serviceAccountKey.json` is never committed. Delete after use. All secrets via
environment variables or Firebase App Settings only.

---

## 2. Push Notifications (FCM / iOS)

### One push path only — native SW push event
All notifications must be handled by a single `self.addEventListener('push', ...)`
in `firebase-messaging-sw.js`. Do NOT import or initialize Firebase compat SDK in
the SW — it registers its own push listener and causes double delivery. (DEC-115,
ISS-113)

### Remove `onMessage` foreground handler entirely
With the native push event as the sole path, remove `onMessage` from
`usePushNotifications.js` entirely. Any foreground handler creates a second
delivery path. iOS suppressing foreground notifications is OS behavior, not a bug.

### VAPID key: copy/paste only, never retype
A single wrong character in the VAPID key causes push to silently fail — no error
is thrown, FCM returns success, but the device never receives anything. Always
paste the key; never transcribe from a screenshot or memory. (ISS-070)

### Token refresh on every mount, replace not append
Run `checkAndRefreshToken` on every component mount. If the token has changed or
the array has more than one entry, replace the entire `fcmTokens` array with
`[freshToken]`. Never use `arrayUnion`. Accumulation of stale tokens = users
stop receiving pushes with no visible error. (DEC-116, ISS-088, ISS-101)

### FCM success ≠ delivered
FCM returns a success response even when iOS silently drops the notification.
If a user reports missing notifications, the problem is almost always a stale
token or a SW registration issue — not an FCM send failure.

### birdieNotify.js is independent from push.js
`birdieNotify.js` has its own send function — it does not share logic with
`push.js`. Changes to one do not propagate to the other. Always update both if
changing notification send behavior. (DEC-111)

### Birdie dedup key must be `{scoreId}_h{holeIndex}`
Cloud Functions v2 fires at-least-once. Always guard birdie notifications with
an idempotency key. Use `{scoreId}_h{holeIndex}` as the dedup key. Do not use
`event.id` — it is not stable across retries.

### `notifyRoundLocked` must be called before the idempotency guard
In `prizes.js`, the round-locked notification must fire before any
`prizesCalculatedAt` guard — otherwise a re-lock after a crash skips the
notification silently. (DEC-087)

### `generatePairings` push: delay 2 seconds after `batch.commit()`
The pairings notification must be delayed ~2s after the Firestore batch commits.
Without the delay, clients haven't yet received the updated data when the
notification arrives, and the app shows stale state. (DEC-091)

---

## 3. iOS PWA / Mobile UI

### Never use floating modal overlays for forms
The `.modal-overlay` / `.modal` pattern becomes unscrollable on iOS PWA
standalone mode. The Save button can be hidden permanently behind the fixed
bottom nav with no way to scroll to it. Body-level `position: fixed` (for scroll
lock) changes the containing block of other fixed elements like `.bottom-nav`,
breaking their viewport-relative positioning.

**Always use inline forms instead:** show/hide with a `showForm` state boolean,
hide the list, show the form in its place — same pattern as `NotesTab`,
`AnnouncementForm`, `DiningEventForm`. (DEC-136, ISS-136)

### `window.confirm()` is silently blocked in PWA standalone
iOS PWA standalone mode blocks `window.confirm()` without throwing — it returns
`false` silently. Always use inline confirmation UI (show/hide a confirm panel).
(ISS-063)

### `position: fixed` elements need explicit z-index and safe-area padding
Fixed elements on iOS (bottom nav, modals, sticky footers) require:
- Explicit `z-index` — don't rely on stacking order
- `padding-bottom: calc(env(safe-area-inset-bottom) + Npx)` for anything
  near the bottom of the screen
- `dvh` units for viewport height — more reliable than `visualViewport` JS

### `position: sticky` is unreliable in flex overflow containers
Use `position: fixed` for sticky headers/footers in scrollable containers.
`sticky` does not behave as expected inside flex containers with `overflow: auto`.

### ResizeObserver for dynamic fixed header height
Use `el.offsetHeight` (not `contentRect.height`) in a `ResizeObserver` to drive
`paddingTop` on scrollable containers below fixed headers. This handles notches,
dynamic content, and orientation changes correctly. (DEC-077)

### `dvh` + `position: fixed` for modal bounds
When a modal must be bounded to the visible viewport (not the full page height),
use `height: 100dvh` and `position: fixed`. Avoid `100vh` on iOS — it includes
the browser chrome.

### Chrome DevTools at 393×852 is the primary testing viewport
Always verify layout at iPhone 15 Pro dimensions before shipping any UI change.
The bottom nav, safe area insets, and fixed element positioning all behave
differently at this size than on desktop.

---

## 4. PWA / Service Worker

### `injectManifest` + `rollupFormat: 'iife'` is the only working SW strategy
With Firebase + iOS, the `injectManifest` strategy and `rollupFormat: 'iife'` in
the Vite PWA plugin config are required. Other strategies cause SW scope
conflicts that break push notification delivery on iOS.

### Cache headers must be set explicitly in `firebase.json`
- `index.html` → `no-cache` — browsers will serve stale HTML otherwise
- SW files → `no-cache` — stale SW = invisible deploys
- `**/*.@(js|css)` → `max-age=31536000, immutable` — hashed filenames are safe

### Always build before deploy
Run `npm run build` before every `firebase deploy --only hosting`. Never deploy
a stale build. Functions require a separate deploy: `firebase deploy --only functions`.

---

## 5. React / Component Patterns

### Format-driven, never round-number-hardcoded
All behavior that varies by round format must be driven by constants or Firestore
data — never by `if (roundNumber === 3)`. A format change in Firestore should
be the only thing needed to change behavior. (DEC-103, DEC-125)

### `useMemo` declaration order matters
In `useLeaderboard.js`, `scrambleResults` must be declared before `r3Leaderboard`
because `r3Leaderboard` depends on it. Wrong order = `ReferenceError` at runtime.
Always declare derived state in dependency order.

### `hasDefaulted` ref pattern for one-time default tab selection
When a screen should default to a tab based on app state (e.g. active round) but
then respect user tab changes, use a `useRef(false)` flag (`hasDefaulted`) that
fires once on mount and never again. Do not use `useEffect` with state-derived
deps — it re-fires on every re-render. (DEC-123)

### `onSnapshot` with `includeMetadataChanges: true` for stale cache detection
For critical real-time data (e.g. outing status), add `includeMetadataChanges: true`
to the `onSnapshot` call. Without it, iOS PWA may serve cached Firestore data
after the app is backgrounded and reopened. (ISS-103)

### `displayName` fallback must be `''`, not `user?.email`
In `useAuth.jsx`, the displayName fallback must be an empty string. Using
`user?.email` as a fallback causes the email address to flash briefly on the
screen during auth state resolution. (ISS-118)

### Admin score entry uses `selectedFoursomeId`, not role-gating the UI
Admin score entry bypasses the normal foursome lookup with a `selectedFoursomeId`
state and a group picker UI. The `isAdmin` check is used to show the picker, not
to change the scoring logic itself. (DEC-113, ISS-108)

---

## 6. Data Integrity

### `yearsPlayed` is always computed, never stored
`yearsPlayed = currentYear - firstYear + 1`. Never write this to Firestore.
Only `firstYear` is stored (set by admins). Storing a derived value creates
drift between the stored value and reality over time.

### Age group field must be `ageGroup`, not `ageCategory`
The correct field name is `ageGroup`. Any document with `ageCategory` instead
will be silently excluded from age-group filtering because the field name check
fails. Audit player docs if age filtering behaves unexpectedly. (ISS-135)

### `normaliseAge()` returns `'U50'` / `'50+'` — not plain strings
The age group normalisation function returns these two exact values. String
comparisons against other formats (e.g. `'under50'`, `'over50'`) will silently
fail. Always use `normaliseAge()` output as the canonical values.

### `scrambleResults` must expose `playerIds` and `holes`
`prizes.js` depends on both fields being present in the `scrambleResults` shape.
If either is missing, scramble prize calculation silently fails. (ISS-081, DEC-085)

### `teeTimes` field: always split with `/[\n,]+/`
Tee time strings in Firestore may use newlines, commas, or both as separators.
Always split with the regex `/[\n,]+/` — never split on a single character.

### `AdminFoursomes` save must use `getDocs`, not `onSnapshot`
The save path in `AdminFoursomes` reads existing docs before writing. Use
`getDocs` for the pre-save read — not `onSnapshot`. A snapshot listener in a
save handler creates race conditions. (ISS-116)

### R2/R4 foursomes save must write both foursomes and matches docs
A save that only updates foursomes leaves match docs stale. Both must be written
in a single batch. (ISS-116, DEC-118)

---

## 7. Deployment & Environment

### PowerShell: no `&&` chaining, always sequential commands
PowerShell does not support `&&`. Run commands on separate lines or use `;`.
Never chain install + build + deploy in a single command.

### Always `--legacy-peer-deps` — no exceptions
`npm install` without `--legacy-peer-deps` fails on this project due to
vite-plugin-pwa peer dependency conflicts. This flag is always required.

### Rules-only deploy when only Firestore rules changed
`firebase deploy --only firestore:rules` — no rebuild needed. Faster and
reduces risk of accidentally deploying a stale build. Only use the full
`firebase deploy --only hosting` when source files changed.

### Cloud Function public access must be set manually after every new function deploy
After deploying a new Cloud Function, go to Cloud Run console and set public
access manually. Firebase does not do this automatically for new functions.

### `.env` and `local.settings.json` are always gitignored — confirm before first commit
Never commit secrets. The `functions/.env` file is never committed. Always
confirm `.gitignore` before the first `git push` on any new machine or project.

### Laptop setup: copy `.env` manually from desktop
`.env` is gitignored — it is not in the repo. On first use of the laptop, copy
it manually from the desktop. Run `npm install --legacy-peer-deps` before any
build or dev server.

### `outing_2026` — underscore, always
The outing ID uses an underscore, not a hyphen. A hyphen creates a different
document that silently coexists with no data. Check constants if outing data
appears to be missing entirely.

---

## 8. Debugging Protocol

### Do not jump to conclusions — rule out the obvious first
Before forming a complex hypothesis:
1. Read the full error message
2. Check the most obvious cause (typo, wrong import path, missing field, wrong doc ID)
3. Verify Firestore rules are not silently blocking the query
4. Read the relevant source files fully before diagnosing logic bugs
5. State what has already been tried

### One fix at a time
Never stack multiple speculative changes. Test one fix, confirm it, then move to
the next. Stacked changes make it impossible to know which one fixed (or broke)
the behavior.

### Firestore rules failures masquerade as missing data
When a query returns empty and you expect data, check Firestore rules before
checking the query logic. An empty `[]` result with no error is almost always a
rules failure. (ISS-140, DEC-139)

### GCP Logs Explorer: `spanId` filtering for Cloud Function isolation
When debugging a Cloud Function, filter by `spanId` in Logs Explorer to isolate
a single invocation. Function logs from multiple concurrent invocations interleave
and are very hard to read without this filter.

### Silent failures to check first
- Push not delivering → stale FCM token or SW registration issue (not a send failure)
- Collection query returns `[]` → Firestore rules blocking a broad read
- Modal button unreachable → iOS PWA scroll/fixed-element containment issue
- Leaderboard age filter not applying → `ageCategory` vs `ageGroup` field mismatch
- Prize calculation not running → `notifyRoundLocked` called after idempotency guard

---

## 9. Security

### No sensitive values in MD files, chat, or screenshots
API keys, tokens, passwords, connection strings, and doc IDs that act as
credentials must never appear in documentation, chat responses, or screenshots.
Use variable name references only: `ANTHROPIC_API_KEY`, not the value.

### VAPID key and all push credentials: text only
Never read a VAPID key or FCM config value from a screenshot. Always require
copy/paste of the raw text. One wrong character = silent failure with no error.

### Firestore write rules unchanged when opening reads
When opening a read rule to fix a broad query failure (e.g. payments collection),
always verify the write rule is unchanged. Document the decision in DecisionLog
with the specific fields being exposed and the privacy tradeoff accepted.

---

## 10. Accessibility

### `user-scalable=no` is an accessibility violation — do not use
Use `maximum-scale=5.0` in the viewport meta tag. Pinch zoom must always be
available. (DEC-097, ISS-090)

### Leaderboard numbers use `var(--font-ui)`, not mono
Leaderboard score cells (`.lb-table__seed-num`, `.lb-table__score`,
`.lb-table__tpar`) use `var(--font-ui)`. Mono font is for admin data-entry
screens only. Never use bare `'monospace'` — always use `var(--font-mono)`.
(DEC-128, ISS-130)

### Login screen text contrast: `rgba(255,255,255,0.75)` on dark green
Secondary text on the login screen must meet contrast requirements. Use
`rgba(255,255,255,0.75)` for subdued text on the dark green background.
(DEC-099, ISS-092)