# TNPL — Project Roadmap

**Last updated:** 2026-09-22

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
- [x] Firestore security rules (all 8 collections, validated via dry-run)
- [x] Page shells: Home, Players, Rankings, Matches, Admin, SignIn — route guards by role
- [ ] `seasonEnrollment` opt-in UI (collection exists in rules/model; UI not built yet)
- [ ] Availability collection UI
- [ ] Pairing engine (Cloud Function)
- [ ] Live scoring UI (matchGroups.sets, editable until locked)
- [ ] Lock Week admin function (Elo calc + players.currentElo update)
- [ ] Change-request create + admin approve/deny UI

## Phase 3 — Migration
- [ ] Import current player roster + Season 3 starting Elos from TNPL_MAIN.xlsx
- [ ] Seed initial `seasonEnrollment` from known opt-ins

## Phase 4 — Polish / launch [PLACEHOLDER]
- [ ] Offline support (as in UP Golf PWA)
- [ ] Editable K-factor / Elo variables in admin UI
