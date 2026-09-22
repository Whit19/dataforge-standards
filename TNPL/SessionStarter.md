# TNPL — Session Starter

**Project:** Thursday Night Paddle League (TNPL) PWA
**Status:** In progress
**Last updated:** 2026-09-22

## One-line description
A PWA for Thursday Night Paddle League: live scoring, member sign-up, Elo rankings, and weekly match creation/viewing — similar to the UP Golf and Club Golf PWAs.

## Current status
Phase 2 core build is underway. Done: app scaffold, Firebase project (`tnpl-pwa`) live with Firestore + Google/Email Link auth providers enabled, auth wiring (including the `playerLinks` reverse-index pattern — see TP-014), full Firestore security rules for all 8 collections, and page shells (Home/Players/Rankings/Matches/Admin/SignIn) with role-based route guards.

## Next priorities
1. Pairing engine (Cloud Function) — the meaty logic piece, see TechnicalArchitecture.md's pairing algorithm section.
2. Live scoring UI (`matchGroups.sets`, editable until locked).
3. Lock Week admin function (Elo calc + `players.currentElo` update).
4. `seasonEnrollment` opt-in UI (collection/rules already exist, no UI yet).
5. Change-request create + admin approve/deny UI.

## Key decisions so far
See DecisionLog.md — TP-001 through TP-014. Notably: full weekly loop is v1 scope (TP-004), scores are editable until an explicit admin "Lock Week" action (TP-011/012), and season enrollment is modeled separately from the persistent roster (TP-009).

## Open questions
- Firebase project ID/naming: resolved — `tnpl-pwa`.
- Whether staff should see more/less than the read-only weekly schedule: working assumption stated in TechnicalArchitecture.md, not yet challenged by Tom.
