# TNPL — Project Roadmap

**Last updated:** 2026-09-22

## Phase 0 — Kickoff (in progress)
- [x] Define project name, stack, repo name
- [x] Create Notion project row + link services
- [ ] Create GitHub repo (private)
- [ ] Commit standard docs to dataforge-standards/TNPL/
- [ ] Create Claude Project and paste in Project Instructions
- [ ] Confirm .gitignore and no committed secrets
- [ ] If applicable, copy pwa-firebase-rules skill into .claude/skills/

## Phase 1 — Discovery / Requirements [PLACEHOLDER]
- [ ] Turn existing Excel Elo workbook + manual process notes into formal requirements
- [ ] Decide data model (players, matches, weeks, sets, Elo history, availability)
- [ ] Decide availability-collection flow (replacing Mailmeteor + Google Form)
- [ ] Decide match-pairing algorithm approach

## Phase 2 — Core build [PLACEHOLDER]
- [ ] Firebase project setup (Auth, Firestore, Hosting)
- [ ] Member sign-up
- [ ] Weekly match creation + live scoring
- [ ] Elo calculation engine (port existing formulas — see TechnicalArchitecture.md)
- [ ] Season rankings / history views

## Phase 3 — Automation [PLACEHOLDER]
- [ ] Automated weekly availability emails
- [ ] Automated pairing suggestions (Elo-diff constrained)
- [ ] Change-request workflow (player requests, Tom approves)

## Phase 4 — Polish / launch [PLACEHOLDER]
- [ ] Offline support (as in UP Golf PWA)
- [ ] Season 3 migration from Excel
