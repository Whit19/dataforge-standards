# TNPL — Project Roadmap

**Last updated:** 2026-10-05

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
- [x] Roster tab redesign: Active / Pending / To Invite / Not Active, replacing the old three-tab version that put every never-invited player in Active (ISS-014, TP-041)
- [x] Public, admin-editable "Rules & league info" page (`/info`, `leagueInfo/main`) + admin editor (`/admin/info`); Home's initials link replaced with an "ⓘ Info" pill (TP-042)
- [x] Public "Request to join" page (`/join`), the existing join-request form extracted to a shared component (TP-042)
- [x] iPhone install steps updated for iOS 26 Safari (••• before Share) across the install gate, install-help sheet, and invite email
- [x] Availability collection UI — existing questions plus dinner / golf-sim questions (writes `socialPlans`); `weekId`/`playerId` fields on availability docs (ISS-005 resolved); Monday-morning email + Tuesday reminder; one-tap email answers; admin response tracker (TP-044)
- [x] Weekly dinner / golf-sim section UI (admin + staff + players) — shared `SocialPlanEditor` on both Matches and Home
- [x] Player Matches page (compact style) + score entry (`matchGroups.sets`, per-set lineups from `setLineups`; any-order set entry; editable until locked) (TP-045)
- [x] Lock Week admin function (Elo calc reading each set's teams from `setLineups` + `players.currentElo` update), transactional lock/unlock, Tuesday reminder + Wednesday auto-lock, admin "Scores and lock" screen (`/admin/scores`) (TP-046, TP-047, TP-048)
- [x] Rankings / History / Season summary pages, computed on-device from `eloHistory` via `src/lib/standings.js` (TP-051)
- [x] Change-request admin approve/deny UI: player request form, withdraw, `/admin/changes` page + dashboard tile, approval via `PairingDraft.jsx`'s `PlayerSheet` inside the same transaction as the pairing edit (TP-006, TP-049, TP-050) — "Text Tom" links now stand in only outside the request window or when no League contact is set
- [x] Settings: Elo/pairing values, default + per-week time slots (with per-slot courts), skip a week, admins, golf-sim link, League contact, and an uploaded menu PDF in Firebase Storage (TP-053, TP-054); new admin `Season.jsx` screen
- [x] Week extras: a per-week dinner special and an optional yes/no question, in the app, the Monday email, and the email answer page, with a tracker summary (TP-055)
- [x] Players tab + real contact privacy: server-maintained `directory`/`playerRatings` collections, a listing rule (active + signed in + opted in; staff once active), hide-my-phone/email switches, a League contact for "Text Tom," and `players` reads locked to admin + self (TP-056, TP-057)
- [x] Full emulator dress rehearsal of the weekly loop (`functions/scripts/runWeeklyLoop.js`), 74/74 PASS (TP-059); found and fixed a real bug (`directorySync` crashing on a `players` doc missing `seasonStartElo`, ISS-027/TP-058)
- [x] Batch C — weather: forecast library + tests, a "Weather location" Settings row, the Home weather card (coldest-slot layer badge, wind, rain/snow) (TP-060)
- [x] Cancel a night or slot: per-slot `matchGroups.status: 'cancelled'` after pairings are sent, admin `cancelSlots`/`restoreSlot` callables with emails, Lock/Elo and pairing-history handling, Home/Matches banners and cancelled-card states, admin `/admin/cancel` screen (TP-061, TP-062)
- [x] Automation watchdog: hourly check that `weeklyAutomation`'s steps actually ran, plus a crash alert on the automation itself (TP-066)
- [x] Roster "Invite selected" in select mode, reusing `sendInvites` (ISS-035) — used for the Oct 3 invite send (two days ahead of the planned Oct 5) instead of un-holding "Invite all" (TP-069)
- [x] Invite-day fixes: decline link no longer records on GET (ISS-038, TP-070); a single-player re-invite can reach a declined player who never signed in (ISS-039, TP-071); "Resend invite" shows its real result inline (ISS-040, TP-072)
- [x] Found-by-real-tester fixes: players' Rankings/History permission-denied (ISS-028, ISS-029), `useWeekMatches` hiding real errors (ISS-030), History opening on the clicked player (ISS-033), iOS weather-coordinate keyboard (ISS-034), installed PWA not picking up new deploys + a build-time "Updated" stamp (ISS-032)
- [x] Staff can view Rankings, History and Season summary — reverses TP-016 (TP-063)
- [x] Part of Batch D message wording: invite email mentions Chrome on Android, "partner with each player once," View menu moved to a header pill, dinner/golf-sim plans bolded consistently and matched by player id (ISS-036)
- [x] Pending-tab reminder email (`sendInvites` `mode: 'reminder'`, two variants, "Send a test to me"), Roster "Select all" (closes the rest of ISS-035) (TP-073)
- [x] Admin can set a player's season choice for them, including before they've signed in — `setSeasonSignup` admin override, `classify`/`notActiveReason` fixed to honor it, edit sheet "This season" section (TP-074)
- [x] Duplicate players: `mergeDuplicatePlayer` admin callable, `resolveJoinRequest`'s `linkToPlayerId`, shared `likelySameName` matcher on both the edit sheet and the Requests tab (ISS-044, TP-075)
- [x] Reminder-send timeout incident fixed: `sendInvites` explicit 540s server + client `httpsCallable` timeout, a 24h `reminded_recently` duplicate-send guard with a server-only `force` bypass, a friendly timeout message, "Reminded {date}" on Pending rows, and skip-count/button wording changes (ISS-045, TP-076)
- [ ] Per-slot pairing re-run on change-request approval (deferred, not built — ISS-006)
- [ ] Whole-week cancel before pairings are sent (reserved, not built — see TP-061; would use the existing, still-unused `WEEK_STATUS.CANCELLED`)
- [ ] "Invite all" — deliberately held; "Invite selected" is used instead (TP-039, TP-069)
- [ ] Push notifications, admin progress bar on Home, custom profile fields (rest of Batch D)
- [ ] After Oct 15: edit sheet reads a point-in-time player copy (ISS-041); tab bar floating on Admin, if it recurs (ISS-042); the reminder's choice-blind `not_signed_in` targeting (ISS-043)

## Phase 3 — Migration
- [x] `TNPL_MAIN.xlsx` imported as the initial roster + Season 3 starting Elos (45 players); import-only going forward — Firestore is the source of truth after this (TP-040)
- [x] Season 25-26 history import from the MASTER sheet (17 weeks, 68 matches, 251 eloHistory docs; players no longer on the roster are historical-name-only references via `directory`) (TP-052)

## Phase 4 — Polish / launch [PLACEHOLDER]
- [ ] Offline support (as in UP Golf PWA)
- [x] Editable K-factor / Elo variables (and pairing flag threshold / slot fairness tolerance) in admin Settings; admin season-creation screen shows the defaults (100 / 25)
- [ ] Season close: carry-over into next season using the stored (but not yet consumed) `carryOverFactor`/`carryOverSetsPivot`, and the "next season starting Elos" link on Season summary
