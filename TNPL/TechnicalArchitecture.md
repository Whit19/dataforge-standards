# TNPL — Technical Architecture

**Last updated:** 2026-09-22

## Stack
- Frontend: React + Vite
- Backend / data: Firebase — Firestore (data), Firebase Auth (member sign-up), Firebase Hosting (deploy), Cloud Functions (pairing engine, Elo calc)
- Repo: https://github.com/Whit19/TNPL (private)
- Local folder: `C:\Dev_Projects\TNPL`
- Docs: `C:\Dev_Projects\dataforge-standards\TNPL\` (cloned locally)

## Scope for v1
Full weekly loop, built now: availability collection → pairing → live scoring → Elo recalculation. (Decided 2026-09-22 — see DecisionLog TP-004.)

## Time-slot / court structure
Confirmed from live workbook (`Fall_Paddle_2026_Session_2_Copy.xlsx`): 3 fixed time slots per Thursday, 2 groups (courts) per slot, 4 players per group.
- 6:00 PM — Group 1, Group 2
- 7:15 PM — Group 3, Group 4
- 8:30 PM — Group 5, Group 6

This is not necessarily fixed forever — `courtsAvailable` is stored per slot per week so it can flex.

## Data model (Firestore)

```
players/{playerId}
  name, email, phone, currentElo, active, isAdmin

seasons/{seasonId}
  startDate, endDate, kFactor, startingElos: { playerId: elo }

weeks/{weekId}
  seasonId, date, status: draft | availability_open | matches_set | in_progress | complete
  timeSlots: [{ id, label: "6:00 PM", courtsAvailable: 2 }, ...]

availability/{weekId}_{playerId}
  canPlay, canPlayTwo, blockedSlotIds[], preferredSlotIds[], notes, respondedAt

matchGroups/{weekId}_{groupId}
  slotId                              // which timeSlot this group belongs to
  matchNumber: 1 | 2                  // which of a player's up-to-2 matches this is, for players playing twice
  team1: [playerId, playerId], team2: [playerId, playerId]
  sets: [{team1Score, team2Score}]    // winner's score only, matches existing convention
  status: scheduled | in_progress | complete

changeRequests/{requestId}
  weekId, matchGroupId, playerId
  type: swap_out | time_change
  reason, status: pending | approved | denied
  requestedAt, resolvedAt, resolvedBy

eloHistory/{weekId}_{playerId}
  eloBefore, eloAfter, delta
```

No locked-partner/couples constraint — that only applied to a one-off event, not regular Thursday play (confirmed 2026-09-22).

## Elo formulas (ported from the existing Excel system)
- K-factor: 32 (editable per season — future-state requirement)
- Team Elo = average of the 2 players' individual Elos
- Expected score (Team A) = 1 / (1 + 10^((TeamB_Elo − TeamA_Elo) / 400))
- Margin multiplier = 0.6 + 0.16 × (|point_diff| − 1)
- Per-set adjustment = K × margin_multiplier × (actual_result − expected_score), same adjustment applied to both teammates
- End-of-week update: all 3 sets in a match use the player's start-of-week Elo as basis; the 3 set adjustments are summed and applied once per week
- Next-season regression: shrinkage = 0.7 × sets_played / (sets_played + 20); next_start = 1500 + shrinkage × (final_elo − 1500)

## Pairing algorithm (server-side, Cloud Function)
1. Pull the week's availability responses + current Elo for everyone who said yes.
2. Build the candidate pool per time slot, respecting blocked/preferred slots.
3. Within each slot, greedily sweep sorted-by-Elo players into foursomes where max−min Elo ≤ 75 (configurable), backtracking when a player can't be placed without breaking the threshold. Group count per slot is capped by `courtsAvailable`.
4. For players who can play 2 matches, run a second pass across slots so their two groups don't reuse the same partner/opponents where avoidable.
5. Players who can't be placed (odd count, incompatible availability) are surfaced to the admin for manual resolution before groups are published.
6. Re-runnable per-slot when a change request is approved, rather than re-pairing the whole week.

## Change requests
Player flags an issue with their assigned group (swap out / time change) → status `pending` → admin (Tom, `isAdmin: true`) approves or denies from a queue → approval re-triggers pairing for just that slot.

## Known constraints / preferences
- Local dev on Windows 11, VS Code, PowerShell.
- [PLACEHOLDER: Firebase project ID / naming, e.g. `tnpl-pwa` following the `up-golf-pwa` convention]
