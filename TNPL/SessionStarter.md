# TNPL — Session Starter

**Project:** Thursday Night Paddle League (TNPL) PWA
**Status:** In progress
**Last updated:** 2026-09-23

## One-line description
A PWA for Thursday Night Paddle League: live scoring, member sign-up, Elo rankings, and weekly match creation/viewing — similar to the UP Golf and Club Golf PWAs.

## Current status
Phase 2 core build is underway. Done: app scaffold, Firebase project (`tnpl-pwa`) with Firestore + Google/Email Link auth, auth wiring (incl. the `playerLinks` reverse index, TP-014), Firestore security rules for every collection, page shells with role-based route guards, and the **pairing engine** — a pure, tested engine plus the admin-only `generatePairings` callable, verified end-to-end against the local Firebase emulators (see TechnicalArchitecture.md "Pairing engine" and "Local emulator testing"). Nothing is deployed yet; rules and functions exist locally and in the emulator only.

## Next priorities
1. Admin draft/publish screen (groups, flags, overflow/unplaced, slot tallies, **unmet preferred slots**; publish = `pairing_draft` → `matches_set`).
2. Availability form with dinner / golf-sim questions (writes `socialPlans`; also write `weekId`/`playerId` on availability docs).
3. Weekly dinner / golf-sim section UI (admin + staff + players).
4. Live scoring UI (per-set lineups) → Lock Week (Elo reads each set's teams from `setLineups`).
5. `seasonEnrollment` opt-in UI; change-request create + approve/deny UI.

## Key decisions so far
See DecisionLog.md — TP-001 through TP-026. Notably: full weekly loop is v1 scope (TP-004), scores editable until an admin "Lock Week" (TP-011/012), season enrollment separate from the roster (TP-009), partners rotate every set (TP-017), the 75-pt Elo rule became a review flag (TP-018), and pairings land as an admin-only `pairing_draft` first (TP-020).

## Open questions
- **TP-015 (mid-season opt-out) still open.** The engine only reads `optedIn` at run time; publish and change requests will need the decision.
- For two-match volunteers, should the engine prefer back-to-back slots or a gap between matches? The seed run produced both (6:00+7:15 and 6:00+8:30). Undecided.
- Whether staff should see more/less than the read-only weekly schedule: working assumption stated in TechnicalArchitecture.md, not yet challenged by Tom.
