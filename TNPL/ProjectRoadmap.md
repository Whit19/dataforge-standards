# TNPL — Project Roadmap

**Last updated:** 2026-09-23

## Phase 0 — Kickoff (done)
- [x] Repo, Notion, docs, .gitignore, pwa-firebase-rules skill

## Phase 1 — Discovery / Requirements (done)
- [x] Data model, time-slot/court structure, pairing approach, v1 scope
- [x] Auth/roles model, season-enrollment model, staff visibility, score-lock flow

## Phase 2 — Core build [IN PROGRESS]
- [x] Vite/React app scaffold, Firebase SDK init, firestorePaths.js/constants.js, functions/ skeleton
- [x] Firebase project `tnpl-pwa` created; Firestore (production mode) + Google/Email Link providers enabled
- [x] `.env.local` populated
- [x] Auth wiring (src/auth.js) — sign-in, email-to-player lookup, playerLinks reverse index, not_on_roster path
- [x] Firestore security rules (all collections incl. socialPlans / pairingRuns and the new matchGroups shape, validated via dry-run — **not deployed**)
- [x] Page shells: Home, Players, Rankings, Matches, Admin, SignIn — route guards by role
- [x] Pairing engine (`functions/pairing/engine.js`, pure + 20 unit tests) and admin-only `generatePairings` callable — verified end-to-end against the emulators (12 checks pass; unmet preferred slot is an accepted WARN, TP-023)
- [x] Emulator setup + seed/verify script (`demo-tnpl`)
- [ ] Admin draft/publish screen: groups, flags, overflow/unplaced, per-player slot tallies, unmet preferred slots; publish moves `pairing_draft` → `matches_set`
- [ ] Availability collection UI — existing questions plus dinner / golf-sim questions (writes `socialPlans`); write `weekId`/`playerId` fields on availability docs
- [ ] Weekly dinner / golf-sim section UI (admin + staff + players)
- [ ] `seasonEnrollment` opt-in UI (collection exists in rules/model; UI not built yet; mid-season opt-out is TP-015, still open)
- [ ] Live scoring UI (`matchGroups.sets`, now per-set lineups from `setLineups`; editable until locked)
- [ ] Lock Week admin function (Elo calc reading each set's teams from `setLineups` + `players.currentElo` update)
- [ ] Change-request create + admin approve/deny UI
- [ ] Per-slot pairing re-run on change-request approval (deferred, not built)
- [ ] First deploy: rules, functions, hosting (after the first functions deploy, set Cloud Run public invoker access manually)

## Phase 3 — Migration
- [ ] Import current player roster + Season 3 starting Elos from `TNPL_MAIN.xlsx` / Player List (MAIN's S3 ELO column was corrected 2026-09-23; the 10 roster players with no Elo start at 1500)
- [ ] Seed initial `seasonEnrollment` from known opt-ins

## Phase 4 — Polish / launch [PLACEHOLDER]
- [ ] Offline support (as in UP Golf PWA)
- [ ] Editable K-factor / Elo variables (and pairing flag threshold / slot fairness tolerance) in admin UI; admin season-creation screen shows the defaults (100 / 25)
