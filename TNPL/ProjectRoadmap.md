# TNPL — Project Roadmap

**Last updated:** 2026-10-23

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
- [x] Roster tab redesign: Active / Pending / To Invite / Not Active, replacing the old three-tab version that put every never-invited player in Active (TPI-014, TP-041)
- [x] Public, admin-editable "Rules & league info" page (`/info`, `leagueInfo/main`) + admin editor (`/admin/info`); Home's initials link replaced with an "ⓘ Info" pill (TP-042)
- [x] Public "Request to join" page (`/join`), the existing join-request form extracted to a shared component (TP-042)
- [x] iPhone install steps updated for iOS 26 Safari (••• before Share) across the install gate, install-help sheet, and invite email
- [x] Availability collection UI — existing questions plus dinner / golf-sim questions (writes `socialPlans`); `weekId`/`playerId` fields on availability docs (TPI-005 resolved); Monday-morning email + Tuesday reminder; one-tap email answers; admin response tracker (TP-044)
- [x] Weekly dinner / golf-sim section UI (admin + staff + players) — shared `SocialPlanEditor` on both Matches and Home
- [x] Player Matches page (compact style) + score entry (`matchGroups.sets`, per-set lineups from `setLineups`; any-order set entry; editable until locked) (TP-045)
- [x] Lock Week admin function (Elo calc reading each set's teams from `setLineups` + `players.currentElo` update), transactional lock/unlock, Tuesday reminder + Wednesday auto-lock, admin "Scores and lock" screen (`/admin/scores`) (TP-046, TP-047, TP-048) — Monday 6 AM is now the main auto-lock, Wednesday the fallback (TP-085)
- [x] Rankings / History / Season summary pages, computed on-device from `eloHistory` via `src/lib/standings.js` (TP-051)
- [x] Change-request admin approve/deny UI: player request form, withdraw, `/admin/changes` page + dashboard tile, approval via `PairingDraft.jsx`'s `PlayerSheet` inside the same transaction as the pairing edit (TP-006, TP-049, TP-050) — "Text Tom" links now stand in only outside the request window or when no League contact is set
- [x] Settings: Elo/pairing values, default + per-week time slots (with per-slot courts), skip a week, admins, golf-sim link, League contact, and an uploaded menu PDF in Firebase Storage (TP-053, TP-054); new admin `Season.jsx` screen
- [x] Week extras: a per-week dinner special and an optional yes/no question, in the app, the Monday email, and the email answer page, with a tracker summary (TP-055)
- [x] Players tab + real contact privacy: server-maintained `directory`/`playerRatings` collections, a listing rule (active + signed in + opted in; staff once active), hide-my-phone/email switches, a League contact for "Text Tom," and `players` reads locked to admin + self (TP-056, TP-057)
- [x] Full emulator dress rehearsal of the weekly loop (`functions/scripts/runWeeklyLoop.js`), 74/74 PASS (TP-059); found and fixed a real bug (`directorySync` crashing on a `players` doc missing `seasonStartElo`, TPI-027/TP-058)
- [x] Batch C — weather: forecast library + tests, a "Weather location" Settings row, the Home weather card (coldest-slot layer badge, wind, rain/snow) (TP-060)
- [x] Cancel a night or slot: per-slot `matchGroups.status: 'cancelled'` after pairings are sent, admin `cancelSlots`/`restoreSlot` callables with emails, Lock/Elo and pairing-history handling, Home/Matches banners and cancelled-card states, admin `/admin/cancel` screen (TP-061, TP-062)
- [x] Automation watchdog: hourly check that `weeklyAutomation`'s steps actually ran, plus a crash alert on the automation itself (TP-066)
- [x] Roster "Invite selected" in select mode, reusing `sendInvites` (TPI-035) — used for the Oct 3 invite send (two days ahead of the planned Oct 5) instead of un-holding "Invite all" (TP-069)
- [x] Invite-day fixes: decline link no longer records on GET (TPI-038, TP-070); a single-player re-invite can reach a declined player who never signed in (TPI-039, TP-071); "Resend invite" shows its real result inline (TPI-040, TP-072)
- [x] Found-by-real-tester fixes: players' Rankings/History permission-denied (TPI-028, TPI-029), `useWeekMatches` hiding real errors (TPI-030), History opening on the clicked player (TPI-033), iOS weather-coordinate keyboard (TPI-034), installed PWA not picking up new deploys + a build-time "Updated" stamp (TPI-032)
- [x] Staff can view Rankings, History and Season summary — reverses TP-016 (TP-063)
- [x] Part of Batch D message wording: invite email mentions Chrome on Android, "partner with each player once," View menu moved to a header pill, dinner/golf-sim plans bolded consistently and matched by player id (TPI-036)
- [x] Pending-tab reminder email (`sendInvites` `mode: 'reminder'`, two variants, "Send a test to me"), Roster "Select all" (closes the rest of TPI-035) (TP-073)
- [x] Admin can set a player's season choice for them, including before they've signed in — `setSeasonSignup` admin override, `classify`/`notActiveReason` fixed to honor it, edit sheet "This season" section (TP-074)
- [x] Duplicate players: `mergeDuplicatePlayer` admin callable, `resolveJoinRequest`'s `linkToPlayerId`, shared `likelySameName` matcher on both the edit sheet and the Requests tab (TPI-044, TP-075)
- [x] Reminder-send timeout incident fixed: `sendInvites` explicit 540s server + client `httpsCallable` timeout, a 24h `reminded_recently` duplicate-send guard with a server-only `force` bypass, a friendly timeout message, "Reminded {date}" on Pending rows, and skip-count/button wording changes (TPI-045, TP-076)
- [x] Admin availability override: tapping a player's name in Admin → Availability opens an edit sheet to set/correct their availability for a week, even after their own answer window has closed — `saveAvailability` gained an optional `playerId` (TP-077)
- [x] Whole-week cancel, after pairings are sent: cancelling every slot in a week now locks it as `CANCELLED`, Elo-neutral — `cancelSlots` itself does it, no separate callable; `restoreSlot` reverses it back to whatever status the week had before (TP-078)
- [x] Automatic messages (Settings): chronological order, 12-hour clock, and each step's day/time is now editable — `updateSeasonAutomation` recomputes upcoming weeks' send times inside a transaction (TP-079)
- [x] Admin availability shortcut on Home, visible before pairings are sent; fixed a row-spacing regression in the Availability admin list from TP-077 (TP-080)
- [x] Post-Week-2 round (Tom's 7 notes, Oct 23), in build order:
  - [x] Rankings lists every enrolled player in the active season ("Not played yet" rows mixed in by starting Elo); Provisional counts past seasons; "1 match" fix (TP-081, TPI-048)
  - [x] Info hub + Rules / Elo / Menu & Golf Sim / Music sub-pages; members-only `leagueInfo/members` for Music; editor "Shows on" picker and "Add starter Elo text" (TP-082)
  - [x] Home after lock — "Your season" card + last week's results with a per-player Elo column (TP-083, TPI-047); match details page with per-set Elo, linked from every final score table on Home and Matches (TP-084)
  - [x] Monday 6 AM auto-lock; any lock opens the next week right away with the new results + Are-you-in email; Monday nudge to non-responders (skipped when the week opened that same morning); session-break exception (TP-085, TP-086, TP-088)
  - [x] Email recipients: admin copies of every player email, full-schedule emails to staff + admins at send/finalize/changes/cancel (TP-087)
  - [x] Weekly-loop rehearsal extended to 5 weeks (4 + a session break), 112/112
  - [x] Week 2's missed results email sent via a one-time script (TP-089, TPI-049)
- [ ] Tom: assign each existing Info section's "Shows on" (League setup → Top of Info; update its availability line to "Next week opens as soon as this week's results are final…"), review the starter Elo text, and add the real Music content (WiFi, Sonos, amp reset, door code)
- [ ] Decide the Thanksgiving early-open behavior before the Nov 19 lock (TPI-051)
- [ ] Show the Monday auto-lock / open-on-lock behavior in Settings' Automatic messages panel (TPI-050)
- [ ] Re-date the weekly-loop rehearsal before ~Dec 2, 2026 (TPI-052)
- [ ] Optional cleanup: make Rankings use `useSeasonStandings` instead of its own inline copy of the same logic (TP-083)
- [ ] Per-slot pairing re-run on change-request approval (deferred, not built — TPI-006)
- [ ] Whole-week cancel **before** pairings are sent (reserved, not built — a week only ever reaches `CANCELLED` via `cancelSlots`, which requires `matches_set`/`in_progress` first, TP-078)
- [ ] "Invite all" — deliberately held; "Invite selected" is used instead (TP-039, TP-069)
- [ ] Push notifications, admin progress bar on Home, custom profile fields (rest of Batch D)
- [ ] After Oct 15: edit sheet reads a point-in-time player copy (TPI-041); tab bar floating on Admin, if it recurs (TPI-042); the reminder's choice-blind `not_signed_in` targeting (TPI-043)

## Phase 3 — Migration
- [x] `TNPL_MAIN.xlsx` imported as the initial roster + Season 3 starting Elos (45 players); import-only going forward — Firestore is the source of truth after this (TP-040)
- [x] Season 25-26 history import from the MASTER sheet (17 weeks, 68 matches, 251 eloHistory docs; players no longer on the roster are historical-name-only references via `directory`) (TP-052)

## Phase 4 — Polish / launch [PLACEHOLDER]
- [ ] Offline support (as in UP Golf PWA)
- [x] Editable K-factor / Elo variables (and pairing flag threshold / slot fairness tolerance) in admin Settings; admin season-creation screen shows the defaults (100 / 25)
- [ ] Season close: carry-over into next season using the stored (but not yet consumed) `carryOverFactor`/`carryOverSetsPivot`, and the "next season starting Elos" link on Season summary
