# TNPL — Project Roadmap

**Last updated:** 2026-09-22

## Phase 0 — Kickoff (done)
- [x] Repo, Notion, docs, .gitignore, pwa-firebase-rules skill

## Phase 1 — Discovery / Requirements (done)
- [x] Data model, time-slot/court structure, pairing approach, v1 scope
- [x] Auth/roles model, season-enrollment model, staff visibility, score-lock flow

## Phase 2 — Core build [IN PROGRESS]
- [x] Vite/React app scaffold, Firebase SDK init, firestorePaths.js/constants.js, functions/ skeleton
- [x] Firebase project `tnpl-pwa` created (Tom, manual step)
- [ ] Tom: populate `.env.local` with real Firebase config values
- [ ] Tom: enable Google + Email Link providers in Firebase console; Firestore in production mode
- [ ] Auth wiring — sign-in, email-to-player lookup, access-denied path for non-roster emails
- [ ] Firestore security rules (see TechnicalArchitecture.md permission model)
- [ ] `seasonEnrollment` collection + opt-in UI
- [ ] Page shells: Home, Players, Rankings, Matches, Admin (route structure + nav, no full feature logic yet)
- [ ] Availability collection UI
- [ ] Pairing engine (Cloud Function)
- [ ] Live scoring UI (matchGroups.sets, editable until locked)
- [ ] Lock Week admin function (Elo calc + players.currentElo update)
- [ ] Change-request create + admin approve/deny flow

## Phase 3 — Migration
- [ ] Import current player roster + Season 3 starting Elos from TNPL_MAIN.xlsx
- [ ] Seed initial `seasonEnrollment` from known opt-ins

## Phase 4 — Polish / launch [PLACEHOLDER]
- [ ] Offline support (as in UP Golf PWA)
- [ ] Editable K-factor / Elo variables in admin UI
