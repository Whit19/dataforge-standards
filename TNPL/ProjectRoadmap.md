# TNPL — Project Roadmap

**Last updated:** 2026-09-28

## Phase 0 — Kickoff (done)
- [x] Repo, Notion, docs, .gitignore, pwa-firebase-rules skill

## Phase 1 — Discovery / Requirements (done)
- [x] Data model, time-slot/court structure, pairing approach, v1 scope
- [x] Auth/roles model, season-enrollment model, staff visibility, score-lock flow

## Phase 2 — Core build [IN PROGRESS]
- [x] Vite/React app scaffold, Firebase SDK init, firestorePaths.js/constants.js, functions/ skeleton
- [x] Firebase project `tnpl-pwa` created; Firestore (production mode) + Google/Email Link providers enabled; on the Blaze plan (TP-033)
- [x] `.env.local` populated
- [x] Auth wiring — sign-in, email-to-player lookup, `playerLinks` reverse index, not-on-roster path. Sign-in method expanded to an emailed 6-digit code as the primary method (Google remains available with `select_account`) — TP-031
- [x] Firestore security rules for every collection, **deployed**
- [x] Page shells with route guards by role, now on a bottom tab bar (Home / Matches / Rankings / Players / Profile, + Admin when `isAdmin`) — TP-037. Matches/Rankings/Players show a Coming Soon page until built
- [x] Pairing engine (pure + unit tests) and admin-only `generatePairings` callable — verified end-to-end against the emulators
- [x] Emulator setup + seed/verify scripts (`demo-tnpl`)
- [x] Admin pairing-draft screen: groups, flags, overflow/unplaced, slot tallies, unmet preferred slots, compact court-grid layout; admin edit actions (swap/move/bench/replace player, move/create match, undo)
- [x] Publish flow: draft → send to players → finalize, with an admin hold option; weekly automation on a 15-minute Cloud Scheduler check (TP-030); notifications on a later admin edit
- [x] Season setup (`setupSeason` callable, admin UI) — creates a season, its sessions, and every week in one step (Season 26-27 created, TP-029)
- [x] Admin dashboard (season/roster/pairing entry points)
- [x] Roster admin screen: add/edit players, send/re-send invites, decline-invite page, public join-request form + admin approve/deny
- [x] Invite email, v1 then v2 "install-first" (TP-035)
- [x] Player profile page: contact info, contact preference, season sign-up, install-help sheet, sign out
- [x] `seasonEnrollment` opt-in UI (Home's "Are you playing?" card + Profile) — per-session choice (Full / Session 1 / Session 2 / Not this season), TP-029/TP-037
- [x] iPhone install gate — Safari-vs-installed-app sign-in split (TP-036)
- [x] Change-request auto-create on mid-season opt-out from an already-published week (TP-027); admin approve/deny UI for change requests **not built yet**
- [x] Roster signed-in-status fix: `firstSignInAt` as the source of truth, no-downgrade rule on re-invite (TP-038); one-time backfill run
- [x] First deploy: rules, functions, hosting — done; Cloud Run public invoker access set for every callable/HTTP function as it was added
- [ ] **Priority 1 — Roster tab redesign:** "Active" currently includes every never-invited player alongside signed-in players (ISS-014); split into something like Signed in / Invited / Not invited / Requests / Inactive
- [ ] Availability collection UI — existing questions plus dinner / golf-sim questions (writes `socialPlans`); write `weekId`/`playerId` fields on availability docs; Monday-morning email + Tuesday reminder; admin response tracker
- [ ] Weekly dinner / golf-sim section UI (admin + staff + players)
- [ ] Player Matches page (compact style) + score entry (`matchGroups.sets`, per-set lineups from `setLineups`; editable until locked)
- [ ] Lock Week admin function (Elo calc reading each set's teams from `setLineups` + `players.currentElo` update)
- [ ] Rankings / season history page
- [ ] Change-request admin approve/deny UI
- [ ] Per-slot pairing re-run on change-request approval (deferred, not built)
- [ ] "Invite all" — deliberately held until the weekly loop works end to end (TP-039)
- [ ] Push notifications, hide-contact toggles, cancel a week/slot, weather, settings, rules page, admin progress bar on Home, a check that Cloud Scheduler automation actually ran

## Phase 3 — Migration
- [x] `TNPL_MAIN.xlsx` imported as the initial roster + Season 3 starting Elos (45 players); import-only going forward — Firestore is the source of truth after this (TP-040)
- [ ] Season 25-26 history import from the MASTER sheet (players no longer on the roster become historical-name-only references)

## Phase 4 — Polish / launch [PLACEHOLDER]
- [ ] Offline support (as in UP Golf PWA)
- [ ] Editable K-factor / Elo variables (and pairing flag threshold / slot fairness tolerance) in admin UI; admin season-creation screen shows the defaults (100 / 25)
