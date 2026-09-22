# TNPL — Technical Architecture

**Last updated:** 2026-09-22

## Stack
- Frontend: React + Vite
- Backend / data: Firebase — Firestore (data), Firebase Auth (Google + Email Link providers), Firebase Hosting (deploy), Cloud Functions (pairing engine, Elo calc, week-lock)
- Firebase project: `tnpl-pwa` (console: https://console.firebase.google.com/u/1/project/tnpl-pwa/overview)
- Repo: https://github.com/Whit19/TNPL (private)
- Local folder: `C:\Dev_Projects\TNPL`
- Docs: `C:\Dev_Projects\dataforge-standards\TNPL\` (cloned locally)

## Scope for v1
Full weekly loop: availability collection → pairing → live scoring → Elo recalculation. (TP-004)

## League structure
Two seasons per year (e.g. Oct–Dec, Jan–Mar — "season" and "session" are used interchangeably; the Excel history uses "Season 2/3"). The full player roster is persistent across seasons; **opting in to a given season is separate from being on the roster** — not everyone plays every season. (TP-009)

## Time-slot / court structure
3 fixed time slots per Thursday, 2 groups (courts) per slot, 4 players per group.
- 6:00 PM — Group 1, Group 2
- 7:15 PM — Group 3, Group 4
- 8:30 PM — Group 5, Group 6

`courtsAvailable` is stored per slot per week so this can flex.

## Data model (Firestore)

```
players/{playerId}
  name, email, phone, role: 'player' | 'staff', active, isAdmin
  currentElo                            // player role only

seasonEnrollment/{seasonId}_{playerId}
  optedIn: boolean, startingElo, enrolledAt

seasons/{seasonId}
  startDate, endDate, kFactor

weeks/{weekId}
  seasonId, date, status: draft | availability_open | matches_set | in_progress | complete
  timeSlots: [{ id, label: "6:00 PM", courtsAvailable: 2 }, ...]

availability/{weekId}_{playerId}
  canPlay, canPlayTwo, blockedSlotIds[], preferredSlotIds[], notes, respondedAt
  // only meaningful for players with seasonEnrollment.optedIn == true for that week's season

matchGroups/{weekId}_{groupId}
  slotId, matchNumber: 1 | 2
  team1: [playerId, playerId], team2: [playerId, playerId]
  sets: [{team1Score, team2Score}]
  status: scheduled | in_progress | reported | locked

changeRequests/{requestId}
  weekId, matchGroupId, playerId
  type: swap_out | time_change            // time_change = correction AFTER pairing is published;
                                            // a pre-pairing slot preference is just availability, not a request
  reason, status: pending | approved | denied
  requestedAt, resolvedAt, resolvedBy

eloHistory/{weekId}_{playerId}
  eloBefore, eloAfter, delta
```

No locked-partner/couples constraint (TP-008).

## Auth & roles
- Firebase Auth, Google + Email Link (passwordless) providers — covers players without a Google/Gmail account. (TP-010)
- On first sign-in, look up `players` by email. Match found → link `authUid` to that player doc. No match → access denied (invite-only via roster, not open sign-up).
- `role: 'staff'` — club workers who need visibility into the published weekly schedule but are not players: no Elo, no availability, not paired, don't opt into seasons.
- `isAdmin: true` — single admin (Tom) in v1.

## Elo formulas (ported from the existing Excel system)
- K-factor: 32 (editable per season)
- Team Elo = average of the 2 players' individual Elos
- Expected score (Team A) = 1 / (1 + 10^((TeamB_Elo − TeamA_Elo) / 400))
- Margin multiplier = 0.6 + 0.16 × (|point_diff| − 1)
- Per-set adjustment = K × margin_multiplier × (actual_result − expected_score), same adjustment applied to both teammates
- Applied once per week at lock time (all 3 sets use start-of-week Elo as basis), not per-set as scores come in
- Next-season regression: shrinkage = 0.7 × sets_played / (sets_played + 20); next_start = 1500 + shrinkage × (final_elo − 1500)

## Scoring & lock flow
- `matchGroups.status`: `scheduled` → `in_progress` → `reported` (a score has been entered; still editable) → `locked` (immutable)
- Either player in a match group can write `sets` any time status is `in_progress` or `reported` — **last write wins**, so an incorrect entry is easy to correct until locked. (TP-011)
- **Lock Week** — admin-only Cloud Function, one action for the whole week: locks every `matchGroups.sets`, computes Elo adjustments, writes `eloHistory`, updates `players.currentElo`. Scores are never used to compute Elo until this runs. (TP-012)

## Pairing algorithm (server-side, Cloud Function)
1. Pull the week's availability responses (from players with `seasonEnrollment.optedIn == true`) + current Elo.
2. Build the candidate pool per time slot, respecting blocked/preferred slots.
3. Within each slot, greedily sweep sorted-by-Elo players into foursomes where max−min Elo ≤ 75 (configurable), backtracking as needed. Group count per slot capped by `courtsAvailable`.
4. For players playing 2 matches, second pass avoids reusing the same partner/opponents where possible.
5. Unplaceable players surfaced to admin for manual resolution before groups are published.
6. Re-runnable per-slot when a change request is approved.

## Change requests
Player flags an issue with their assigned group (swap out / time change) → `pending` → admin approves/denies → approval re-triggers pairing for just that slot.

## Pages (v1)
Home (this week's status/dashboard) · Players (roster + current Elo) · Rankings (season standings) · Matches (weekly groups, live scoring) · Admin (change-request queue, Lock Week action) — staff see Home/Matches only, read-only.

## Known constraints / preferences
- Local dev on Windows 11, VS Code, PowerShell.
- Firebase config values (`VITE_FIREBASE_*`) go directly into `.env.local` by Tom — never relayed through chat.
