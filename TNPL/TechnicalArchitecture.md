# TNPL — Technical Architecture

**Last updated:** 2026-09-22

## Stack
- Frontend: React + Vite
- Backend / data: Firebase — Firestore (data), Firebase Auth (Google + Email Link providers), Firebase Hosting (deploy), Cloud Functions (pairing engine, Elo calc, week-lock)
- Firebase project: `tnpl-pwa` (console: https://console.firebase.google.com/u/1/project/tnpl-pwa/overview)
- Repo: https://github.com/Whit19/TNPL (private)
- Local folder: `C:\Dev_Projects\TNPL`
- Docs: `C:\Dev_Projects\dataforge-standards\TNPL\` (cloned locally)
- CLI config: `firebase.json`, `.firebaserc`, `firestore.indexes.json` (minimal, added to validate/deploy rules)

## Scope for v1
Full weekly loop: availability collection → pairing → live scoring → Elo recalculation. (TP-004)

## League structure
Two seasons per year (e.g. Oct–Dec, Jan–Mar — "season"/"session" used interchangeably). Full player roster is persistent across seasons; opting in to a given season is separate from being on the roster. (TP-009)

## Time-slot / court structure
3 fixed time slots per Thursday, 2 groups (courts) per slot, 4 players per group.
- 6:00 PM — Group 1, Group 2
- 7:15 PM — Group 3, Group 4
- 8:30 PM — Group 5, Group 6

`courtsAvailable` stored per slot per week so this can flex.

## Data model (Firestore)

```
players/{playerId}
  name, email, phone, role: 'player' | 'staff', active, isAdmin
  currentElo                            // player role only
  authUid                               // set on first sign-in once matched

playerLinks/{authUid}
  playerId
  // Reverse index, auth UID -> playerId. Firestore security rules can only
  // do get() by a known document path, not arbitrary queries — this is what
  // lets isAdmin()/isPlayerRole() rules helpers resolve "which player is the
  // caller" from request.auth.uid alone. Written by auth.js alongside the
  // authUid field on the matched player doc. (TP-014)

seasonEnrollment/{seasonId}_{playerId}
  optedIn: boolean, startingElo, enrolledAt

seasons/{seasonId}
  startDate, endDate, kFactor

weeks/{weekId}
  seasonId, date, status: draft | availability_open | matches_set | in_progress | complete
  timeSlots: [{ id, label: "6:00 PM", courtsAvailable: 2 }, ...]

availability/{weekId}_{playerId}
  canPlay, canPlayTwo, blockedSlotIds[], preferredSlotIds[], notes, respondedAt

matchGroups/{weekId}_{groupId}
  slotId, matchNumber: 1 | 2
  team1: [playerId, playerId], team2: [playerId, playerId]
  sets: [{team1Score, team2Score}]
  status: scheduled | in_progress | reported | locked

changeRequests/{requestId}
  weekId, matchGroupId, playerId
  type: swap_out | time_change
  reason, status: pending | approved | denied
  requestedAt, resolvedAt, resolvedBy

eloHistory/{weekId}_{playerId}
  eloBefore, eloAfter, delta
```

No locked-partner/couples constraint (TP-008).

## Auth & roles — implemented
- Firebase Auth, Google + Email Link (passwordless) providers. (TP-010)
- `src/auth.js`: `signInWithGoogle()`, `sendSignInLinkToEmail()` / `completeEmailLinkSignIn()` (handles the cross-device case — link opened without a stored email locally surfaces a `needs_email` state), `lookupPlayerByEmail()`.
- On successful auth: match found in `players` by email → `authUid` written to that player doc + `playerLinks/{authUid}` reverse-index doc created. No match → user is signed back out, `not_on_roster` state returned (invite-only via roster, no dangling half-authenticated session).
- `role: 'staff'` — club workers with schedule visibility only, no Elo/pairing/availability/season enrollment.
- `isAdmin: true` — single admin (Tom) in v1.

## Firestore rules — implemented
`firestore.rules` covers all 8 collections (including `playerLinks`) per the permission model below. Helpers `isSignedInPlayer()` / `isAdmin()` / `isPlayerRole()` centralize the `playerLinks` lookup rather than repeating it per rule. Validated via `firebase deploy --only firestore:rules --dry-run`.

| Collection | Read | Write |
|---|---|---|
| `players` | any signed-in member | admin only, except a player may edit their own `phone` |
| `playerLinks` | own doc only | written only by the app's auth flow (not general client writes) |
| `seasonEnrollment` | any signed-in member | admin only |
| `seasons` | any signed-in member | admin only |
| `weeks` | any signed-in member (staff: read-only) | admin only |
| `availability` | the player themselves + admin (not staff) | the player themselves (own doc only) + admin |
| `matchGroups` | any signed-in member (staff: read-only) | `sets` field only, either player in that group, only while `status` is `in_progress` or `reported` (never `locked`); all other fields admin/pairing-engine only |
| `changeRequests` | the requesting player + admin | create: any player, for themselves; approve/deny: admin only |
| `eloHistory` | any signed-in member (not staff) | never client-writable — Elo Cloud Function only |

## Elo formulas (ported from the existing Excel system)
- K-factor: 32 (editable per season)
- Team Elo = average of the 2 players' individual Elos
- Expected score (Team A) = 1 / (1 + 10^((TeamB_Elo − TeamA_Elo) / 400))
- Margin multiplier = 0.6 + 0.16 × (|point_diff| − 1)
- Per-set adjustment = K × margin_multiplier × (actual_result − expected_score), same adjustment applied to both teammates
- Applied once per week at lock time (all 3 sets use start-of-week Elo as basis)
- Next-season regression: shrinkage = 0.7 × sets_played / (sets_played + 20); next_start = 1500 + shrinkage × (final_elo − 1500)

## Scoring & lock flow
- `matchGroups.status`: `scheduled` → `in_progress` → `reported` (editable) → `locked` (immutable)
- Either player writes `sets` while `in_progress`/`reported` — last write wins. (TP-011)
- **Lock Week** — admin-only Cloud Function: locks every `matchGroups.sets` for the week, computes Elo, writes `eloHistory`, updates `players.currentElo`. (TP-012)

## Pairing algorithm (server-side, Cloud Function — not yet built)
1. Pull the week's availability (players with `seasonEnrollment.optedIn == true`) + current Elo.
2. Build candidate pool per slot, respecting blocked/preferred slots.
3. Greedily sweep sorted-by-Elo players into foursomes where max−min Elo ≤ 75, group count capped by `courtsAvailable`.
4. Second pass for 2-match players avoids repeat partners/opponents where possible.
5. Unplaceable players surfaced to admin.
6. Re-runnable per-slot on change-request approval.

## Pages
Home · Players · Rankings · Matches · Admin (change-request queue, Lock Week) · **SignIn** (route-guard redirect target, added during auth implementation — not a nav item). Staff see Home/Matches only, read-only.

## Known constraints / preferences
- Local dev on Windows 11, VS Code, PowerShell.
- Firebase config values (`VITE_FIREBASE_*`) go directly into `.env.local` by Tom — never relayed through chat.
