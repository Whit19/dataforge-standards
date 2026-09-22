# TNPL — Project Roadmap

**Last updated:** 2026-09-22

## Phase 0 — Kickoff (done)
- [x] Define project name, stack, repo name
- [x] Create Notion project row + link services
- [x] Create GitHub repo (private)
- [x] Commit standard docs to dataforge-standards/TNPL/
- [x] Create Claude Project and paste in Project Instructions
- [x] Confirm .gitignore and no committed secrets
- [x] Copy pwa-firebase-rules skill into .claude/skills/

## Phase 1 — Discovery / Requirements (done)
- [x] Turn existing Excel Elo workbook + manual process notes into formal requirements
- [x] Decide data model (players, matches, weeks, sets, Elo history, availability, change requests)
- [x] Confirm time-slot/court structure (3 slots × 2 courts, from live workbook)
- [x] Decide match-pairing algorithm approach
- [x] Decide v1 scope: full weekly loop, not staged

## Phase 2 — Core build [NEXT]
- [ ] Firebase project setup (Auth, Firestore, Hosting) — needs project ID decision
- [ ] Vite/React app scaffolding
- [ ] Firestore security rules (players, seasons, weeks, availability, matchGroups, changeRequests, eloHistory)
- [ ] Member sign-up / auth
- [ ] Availability collection UI
- [ ] Pairing engine (Cloud Function, per TechnicalArchitecture.md algorithm)
- [ ] Weekly match creation + live scoring UI
- [ ] Elo calculation engine (port existing formulas)
- [ ] Change-request approve/deny flow
- [ ] Season rankings / history views

## Phase 3 — Migration
- [ ] Import current player roster + Season 3 starting Elos from TNPL_MAIN.xlsx

## Phase 4 — Polish / launch [PLACEHOLDER]
- [ ] Offline support (as in UP Golf PWA)
- [ ] Editable K-factor / Elo variables in admin UI
