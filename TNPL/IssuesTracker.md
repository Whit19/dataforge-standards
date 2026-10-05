# TNPL — Issues Tracker

**Last updated:** 2026-10-05

## Open

### ISS-042 — Bottom tab bar floats mid-screen on the Admin dashboard (iPhone installed PWA)
**Status:** Open — can't reproduce yet (logged 2026-10-03)
**Description:** Tom saw the tab bar sitting about two-thirds of the way down the screen, over the "Manage" tiles, with page content above and below it, while scrolled on `/admin`. The same bar was pinned correctly on Roster a minute earlier. Code review ruled out a broken fixed-position containing block: nothing in `src/` sets `transform`, `filter`, `perspective`, `will-change`, `contain` or `backdrop-filter`. There is no inner scroll container, and Admin and Roster use the same `.page` layout. `.tabbar` is plain `position: fixed; bottom: 0` with `env(safe-area-inset-bottom)` padding. Leading hypothesis, unconfirmed: the iOS WebKit bug where a fixed element sticks at a stale position after a native overlay closes. Admin's `<select id="week-picker">` is the only native picker on that screen. The app has no `visualViewport` or `focusout` handling.
**Resolution:** None yet. If it recurs, record the iOS version, whether the week picker was opened first, and whether the phone had been resumed from the background. Any fix must ship before the Oct 14 deploy freeze or wait until after Oct 15.

## Deferred

### ISS-043 — The reminder email ignores an admin-set season choice for a not-signed-in player
**Status:** Deferred (logged 2026-10-05)
**Description:** `resolveReminderTargets` (server) and its client mirror `reminderSkipReason` (`src/lib/rosterStatus.js`) both send the `not_signed_in` reminder variant to any invited, not-signed-in player unconditionally — neither checks `choiceByPlayerId` for that branch, only for the signed-in (`no_season_pick`) branch. So a player an admin has already given a playing season choice to (TP-074), but who still hasn't signed in, keeps getting "the app's ready when you are" reminders even though their season intent is already on record.
**Resolution:** None yet — deliberately left as-is rather than making the client stricter than the server (or vice versa) without changing both together. Low risk in practice: a player set to `none` already moves to Not Active and drops out of the Pending tab entirely, and "Select all" only selects the current tab, so the only players this affects are ones who *should* still be encouraged to install and sign in regardless of their recorded pick. Possible follow-up, not urgent: skip `choice === 'none'` on both `resolveReminderTargets` and `reminderSkipReason` together. Can wait until after the Oct 15 launch.

### ISS-044 — Approving a join request created a duplicate player instead of matching the existing one
**Status:** Resolved (2026-10-05)
**Description:** A player who signed in with a different email than the one already on the roster hit `not_on_roster`, submitted a join request, and `resolveJoinRequest`'s approve path always created a brand-new `players` doc with a guessed starting Elo — the original player kept the real Elo/history, the new copy held the sign-in link. Real example: "Brian C Spahn" duplicated the existing "Brian Spahn." A "just delete the new copy" fix would have stranded the player's sign-in; deleting the original would have lost their Elo.
**Resolution:** New admin callable `mergeDuplicatePlayer` (TP-075) merges the two into one, keeping the original's Elo/history and refusing if either side has any real game history. `resolveJoinRequest` separately gained `linkToPlayerId` so a *new* request can be routed onto an existing player up front, before a duplicate is ever created. Tom merged the Brian Spahn duplicate successfully after Cloud Run "Allow public access" was set for the new function. Cloud Functions count: 42 → 43.

### ISS-041 — The edit sheet reads a point-in-time copy of the player (logged 2026-10-03)
**Status:** Deferred
**Description:** `Roster.jsx` stores the tapped row in `sheet.player` and passes it to `PlayerEditSheet`, so the open sheet's labels (e.g. "Send invite" vs "Resend invite") read a frozen snapshot rather than the live doc. The Roster list is unaffected because it reads the live array. This doesn't change what gets sent: the invite result comes from the server response.
**Resolution:** None yet for the invite label. Post-Oct 15 cleanup candidate: look the player up by id in the live `players` array on each render. Not inherited by the new "This season" section (TP-074, 2026-10-05) — that section was built with its own live `onSnapshot` subscriptions for role/active/choice from the start, specifically to avoid this bug.

### ISS-006 — Per-slot pairing re-run on change-request approval

### ISS-006 — Per-slot pairing re-run on change-request approval
**Status:** Deferred
**Description:** Designed but not built; the callable re-runs the whole week only.

## Resolved

### ISS-007 — Preferred slot had no effect against unplayed slots
**Status:** Resolved (2026-09-23)
**Description:** Found in the emulator run: a preferred slot cost 0, tying with unplayed slots, so slot order decided (Allen Van Scoyk, history s1:0 s2:0 s3:2, preferred s3, landed in s1). A spec gap, not an implementation bug.
**Resolution:** Preferred slot now costs `PREFERRED_SLOT_BONUS = -1`; `assignSlots` pruning shifts per-group costs so the minimum is 0. A preference can still lose to groupmates' rotation cost — accepted (TP-023).

### ISS-008 — Original preferred-slot unit-test fixture couldn't tell a preference from none
**Status:** Resolved (2026-09-23)
**Description:** With only `slotTally {s3:5}`, the player's other slots cost 0 too, so the test proved nothing.
**Resolution:** Fixture gave the other slots small tallies; a new regression test covers the exact emulator case.

### ISS-009 — `TNPL_MAIN.xlsx` S3 ELO column held Season 2 final Elos
**Status:** Resolved (2026-09-23)
**Description:** The MAIN sheet's "S3 ELO" column held Season 2 FINAL Elos, not the regressed Next Season Start values.
**Resolution:** Tom corrected MAIN; verified to match Player List "Season 3 Start Elo" for all 36 Player List names. MAIN has 45 players, 10 with no Elo (`#N/A`): Steve Sewart, Chris Elkendier, Jon Biorkman, John Burkemper, Jon Gould, Nick Gustafson, John Mort, Dave Patzer, Adam Trafton, Dan Wycklendt.

### ISS-010 — Firestore emulator couldn't start (no Java)
**Status:** Resolved (2026-09-23)
**Description:** The emulator needs JDK 21+ (firebase-tools 15.30.2). Temurin 21 was installed via winget but its bin folder wasn't on PATH.
**Resolution:** Set User PATH and `JAVA_HOME` with `[Environment]::SetEnvironmentVariable` (not `setx`, which can truncate a long PATH), then fully restart VS Code — a new terminal tab isn't enough because it inherits VS Code's environment.

### ISS-001 — Two-match volunteers: back-to-back slots or a gap?
**Status:** Resolved (2026-09-28)
**Description:** For `canPlayTwo` players who play twice, the engine didn't prefer adjacent slots or a gap between matches. The seed run produced both (6:00 + 7:15 and 6:00 + 8:30).
**Resolution:** Engine keeps a soft preference for back-to-back slots (`TWO_MATCH_GAP_PENALTY = 2.1`), reviewed against a near-tie case in real data and kept as-is (TP-028).

### ISS-003 — Rules and functions not deployed
**Status:** Resolved (2026-09-28)
**Description:** `firestore.rules` and the `generatePairings` callable existed only locally / in the emulator.
**Resolution:** Deployed. `firestore.rules`, all 17 Cloud Functions, and hosting are live on `tnpl-pwa`; public invoker access was set in the Cloud Run console for each callable/HTTP function as it was added.

### ISS-004 — Mid-season opt-out unresolved
**Status:** Resolved (2026-09-28)
**Description:** See TP-015 in DecisionLog.md. The engine only read `optedIn` at run time; publish and change requests needed the decision.
**Resolution:** See TP-027 — opting out stops weekly emails, and raises a `swap_out` change request instead of silently removing the player if they're already in a published week's match.

### ISS-011 — `sendInvites` downgraded an already signed-in player back to "invited"
**Status:** Resolved (2026-09-28)
**Description:** Re-sending an invite (e.g. to a player who had already signed in) unconditionally overwrote `inviteStatus` to `'invited'` and reset `invitedAt`/the decline token, even though the player had already accepted. Roster then showed them under Invited instead of Active.
**Resolution:** `sendInvites` still sends the email but no longer touches `inviteStatus`/`invitedAt`/the decline token for anyone with `firstSignInAt` set or `inviteStatus: 'accepted'` (TP-038). A one-time backfill script set `firstSignInAt`/`inviteStatus` for the one player whose original link predated that field.

### ISS-012 — Invite-email Home Screen icon didn't match the real app icon
**Status:** Resolved (2026-09-28)
**Description:** The first version of the invite email's "here's what to look for" Home Screen image was a hand-drawn approximation that didn't match the actual installed icon.
**Resolution:** Regenerated the email images directly from the real `apple-touch-icon` file instead of redrawing them, so the email and the actual Home Screen icon match exactly.

### ISS-013 — Incorrect phone numbers from the roster workbook
**Status:** Resolved (2026-09-28)
**Description:** 12 players had incorrect phone numbers carried over from the original import workbook.
**Resolution:** Corrected via a one-time, dry-run-first script (names logged, numbers never logged) run directly against Firestore.

### ISS-015 — Google sign-in popup / Incognito testing quirks (not an app bug)
**Status:** Resolved (not a bug)
**Description:** Chrome can occasionally open the Google sign-in popup minimized or off-screen. Testing sign-in in an Incognito window is unreliable because Incognito blocks third-party cookies the popup flow depends on.
**Resolution:** Not an app defect — a browser/testing-environment quirk. Test sign-in in a normal (non-Incognito) window; if the Google popup seems to do nothing, check for a minimized window before assuming the flow is broken.

### ISS-014 — Roster "Active" tab lists every never-invited player
**Status:** Resolved (2026-09-28)
**Description:** With the three-tab `classify()` rule (TP-038), a brand-new admin-added player who hasn't been sent an invite yet counted as "Active" alongside players who have actually signed in, since "never invited" and "signed in" both landed in the same tab.
**Resolution:** Redesigned to four tabs — Active / Pending / To Invite / Not Active (TP-041). Active now requires both being signed in and having opted into a session; a signed-in player who chose "not this season" lands in Not Active with a note explaining why, distinct from an actual decline.

### ISS-005 — `availability` docs lack `weekId`/`playerId` fields
**Status:** Resolved (2026-09-29)
**Description:** The pairing callable found a week's availability by document-ID prefix `{weekId}_` — a workaround; the seed script already wrote the fields, but the real availability-writing paths didn't yet.
**Resolution:** The weekly-availability build writes `weekId` and `playerId` on every `availability` doc (both the in-app form and the one-tap email-answer path go through server-side callables now, per TP-044).

### ISS-016 — "This match no longer exists" shown after a transient score-page read error
**Status:** Resolved (2026-09-29)
**Description:** `ScoreEntry.jsx`'s Firestore listener treated its `onSnapshot` error callback the same as "read succeeded, no such document," so any transient read error (including one seen while reseeding the emulator, which wipes auth accounts) permanently showed "This match no longer exists" — even though the match existed and was live for everyone else. Per the Firestore SDK's own docs, once the error callback fires the listener is dead for good; no further callbacks arrive.
**Resolution:** A separate `groupError` state now shows a distinct "Couldn't load this match" message, checked before the loading/not-found branches, so a real read error is never conflated with a genuinely missing document. Regression test added for `id` preservation through this path.

### ISS-017 — Home and Matches could show different court numbers for the same match
**Status:** Resolved (2026-09-29)
**Description:** Court number was computed separately in more than one place; an admin edit that left a gap in a slot's court numbering could make Home and Matches disagree about which court a given match was on.
**Resolution:** Court number (position within the slot) is now computed once, in `src/lib/matchSchedule.js`, and consumed by both pages through the shared `useWeekMatches` hook.

### ISS-018 — Home showed stale "Rules — coming soon" and "opens Monday" text
**Status:** Resolved (2026-09-29)
**Description:** Home's header still showed a placeholder "Rules — coming soon" line after the Rules & league info page shipped, and an "Availability opens Monday" line kept showing after availability was actually open for the week.
**Resolution:** Both lines removed/gated correctly as part of the iPhone-walkthrough fixes in the Matches/score-entry batch.

### ISS-002 — TNPL missing from MASTER_CLAUDE_PROTOCOL.md tables
**Status:** Resolved (2026-09-29)
**Description:** TNPL wasn't in the Active Projects table (Section 2) and prefix `TP` wasn't in the DecisionLog prefix table (Section 11) of the root `MASTER_CLAUDE_PROTOCOL.md`.
**Resolution:** Added a TNPL row to both tables (Notion Page column uses the same `[link]` placeholder every other row currently uses).

### ISS-019 — Dinner/golf-sim links built into the wrong component
**Status:** Resolved (2026-09-30)
**Description:** A CC prompt misidentified `SocialPlanEditor` (which only opens from "Change my plans") as the place to add the menu/golf-sim links, so they only showed up when a player opened that editor.
**Resolution:** Extracted a shared `src/components/DinnerGolfLinks.jsx` reading `leagueSettings/main`, used in the actual section link row on Home and Matches; dead reads of `season.menuUrl`/`season.golfSimUrl`/`season.weeklySpecial` (never written anywhere) removed.

### ISS-020 — Matches page hid the dinner/golf-sim card before pairings published
**Status:** Resolved (2026-09-30)
**Description:** `Matches.jsx` returned early before pairings were published for the upcoming week, which hid the dinner/golf-sim section along with everything else below it.
**Resolution:** Added a `DinnerGolfCard` (its own `socialPlans` listener for the upcoming week, no `matchGroups` read) rendered under the "Pairings go out…" notice instead of inside the early-return branch.

### ISS-021 — iOS blocked "View menu" after an awaited `window.open`
**Status:** Resolved (2026-09-30)
**Description:** Calling `window.open` after `await getDownloadURL(...)` is blocked by iOS Safari/installed PWAs, since the user-gesture context is lost by the time the awaited call resolves.
**Resolution:** Fetch the menu's download URL ahead of time and render a real `<a target="_blank">`, so opening it is a direct click on an anchor rather than a script-triggered `window.open` after an async gap (CC_58).

### ISS-022 — Season 25-26 import dry run aborted on a tolerance that was too tight
**Status:** Resolved (2026-09-30)
**Description:** The workbook stores Elo before/after values rounded to 1 decimal but set-level changes at 2 decimals, so after−before could differ from the summed delta by up to 0.09 — wider than the import script's original validation tolerance, correctly aborting the dry run rather than importing silently-wrong data.
**Resolution:** Tolerance widened to ±0.15 to match the source's actual display precision; the stored `delta` field is kept as the exact sum of the set adjustments regardless (TP-052).

### ISS-023 — Starting-Elo lock check wasn't season-scoped
**Status:** Resolved (2026-09-30)
**Description:** `hasLockedMatch` in `functions/roster/roster.js` (and its client mirror in `PlayerEditSheet.jsx`) checked for a locked match in ANY season before refusing a starting-Elo edit — found during the pre-import scoping audit, before importing a second season's history. Left as-is, every returning player would have been blocked from a legitimate 26-27 starting-Elo correction once Season 25-26's locked matches existed.
**Resolution:** Scoped to the active season's weeks only; the client query was replaced by `matchGroupsByWeekIdsQuery`.

### ISS-024 — Dead season fields hid their own replacement
**Status:** Resolved (2026-09-30)
**Description:** `season.menuUrl`, `season.golfSimUrl`, and `season.weeklySpecial` were read in several places but never written by anything, left over from an earlier design.
**Resolution:** Removed; replaced by `leagueSettings.golfSimUrl`/menu-in-Storage (TP-054) and per-week `weeks.dinnerSpecial` (TP-055).

### ISS-025 — Availability email showed "Season Season 26-27"
**Status:** Resolved (2026-09-30)
**Description:** The availability email template prepended "Season " to a season name that already started with "Season," producing a doubled label. No other email had the same pattern.
**Resolution:** Template no longer prepends the word where the season name already includes it.

### ISS-026 — Players page initially exposed every roster person's contact info
**Status:** Resolved (2026-09-30)
**Description:** The first version of the Players page read every `players` doc, so a never-invited or inactive person's phone/email was visible to any signed-in member, not just people who had actually joined.
**Resolution:** Replaced by the `directory`/`listed` model (TP-056) — only active, signed-in, opted-in players (and active staff) are listed with contact info; everyone else is name-only.

### ISS-027 — directorySync crashed on a players doc missing `seasonStartElo`
**Status:** Resolved (2026-10-01)
**Description:** Found by the full emulator dress rehearsal (`functions/scripts/runWeeklyLoop.js`, TP-059). `buildDirectoryEntries()` in `functions/players/directorySync.js` passed `players.currentElo`/`seasonStartElo` straight through to `playerRatings` with no guard. A guest player created by the change-request "approve and replace" flow (`pairing/edit.js`'s `REPLACE_PLAYER`) sets `currentElo` but never `seasonStartElo` — writing `undefined` crashed `syncDirectoryOnPlayerWrite`/`syncDirectoryOnEnrollmentWrite` outright ("Cannot use undefined as a Firestore value"). The player's own `players`-doc write still succeeded; only the background trigger died, so `directory`/`playerRatings` silently went stale for that player with nothing but a server log to show it.
**Resolution:** Both fields now write `null` when missing (TP-058), never a made-up Elo. 3 regression tests added to `functions/players/directorySync.test.js`. A read-only production check found 0 of 67 `role == 'player'` docs missing `seasonStartElo` — no backfill needed, though the check doesn't cover guests (`role == 'guest'`), and change requests only shipped 2026-09-30, so none likely exist yet; re-run `backfillDirectory.js` if one ever does. Deployed: `syncDirectoryOnPlayerWrite`, `syncDirectoryOnEnrollmentWrite`.

### ISS-028 — Players got "permission-denied" on Rankings' pre-season view
**Status:** Resolved (2026-10-02)
**Description:** Found by Pat Foley, the first real non-admin tester (invisible to Tom, since `isAdmin()` passes every rule). Rankings' pre-season view (active season, before Week 1 locks) listened to `sessionEnrollment where sessionId == …` to find who'd opted in — every player's enrollment doc — but the rule only lets a non-admin read their OWN enrollment doc (`enrollmentId.split('_')[1] == myPlayerId()`). Firestore refuses a list query it can't prove is restricted to documents the caller can read, even though the query would have included the caller's own doc among the results.
**Resolution:** Pre-season "opted in" ids now come from `directory` entries with `listed === true` and `role === 'player'` (TP-064) — data already read via `useDirectory()`, no new Firestore read and no rule change. `sessionEnrollment` stays own-doc-only for non-admins. Verified in the emulator as a non-admin player (both the exact denial and the fix), then confirmed fixed on Pat's phone.

### ISS-029 — Players got "permission-denied" on History for the active season
**Status:** Resolved (2026-10-02)
**Description:** Found alongside ISS-028. History built `matchGroups where weekId in [every week of the selected season]` to resolve each match's slot/court for display. For the active, still-mostly-`draft` season, that `in` list included unpublished weekIds; the matchGroups rule's `weekStatus(resource.data.weekId)` `get()` can't be proven safe for an `in` filter containing a non-published week, so Firestore refused the WHOLE query — even though it would have matched zero documents (no matchGroups exist for a draft week at all).
**Resolution:** `weekIds` is now filtered to `complete` weeks only before the query (TP-065); with none, the listener isn't opened at all and the page falls through to its existing "no matches yet" empty state rather than an error. The first emulator diagnosis (seeded with locked weeks) couldn't reproduce this; a second, reproducing production's actual all-draft-season shape, did. Confirmed fixed on Pat's phone; Season summary was not affected by either ISS-028 or ISS-029.

### ISS-030 — `useWeekMatches` hid every `permission-denied`, not just the expected one
**Status:** Resolved (2026-10-02)
**Description:** `useWeekMatches`'s shared `friendlyError` converted ANY `permission-denied` into `null` (no visible error) for every one of its listeners (season, weeks, matchGroups, socialPlans, directory). That was meant for exactly one expected case — a non-admin can't read `matchGroups` before a week is published — but the same blanket swallow also hid the ISS-028/ISS-029 bug on Home and Matches, which made "Matches looks fine" false evidence that those reads were actually succeeding.
**Resolution:** Only the `matchGroups` listener still swallows `permission-denied`; every other listener (season, weeks, socialPlans, directory, and `leagueSettings` via `useLeagueContact`) now surfaces it like any other error, with its code, matching how Rankings/History display errors. Verified in the emulator as a non-admin player that the normal pre-publish state still shows no error (only the one expected, still-swallowed denial).

### ISS-031 — Lock Week wrote a zero-delta `eloHistory` doc for a player with zero sets played
**Status:** Resolved (2026-10-02)
**Description:** Pre-existing in `computeWeekElo` (`functions/elo/elo.js`): its `ensure(playerId)` creates an accumulator entry for every player listed on a group, whether or not they have any saved sets, and `lockWeekInternal`'s write loop wrote an `eloHistory` doc (and a no-op `currentElo` update) for every one of those entries unconditionally. Before per-slot cancel this was latent — a group reaching lock with zero saved sets was already an edge case nobody hit — but became a real, reachable bug once a cancelled group with no saved sets became a normal, expected state (see TP-061/TP-062). Caught by the cancel feature's own regression tests, not by inspection.
**Resolution:** `lockWeekInternal` now filters to `setsPlayed > 0` before writing `eloHistory` or updating `currentElo` (TP-062); `unlockWeekInternal` needed no change, since it already derives everything from the week's actual `eloHistory` docs, never from `matchGroups`/player rosters.

### ISS-032 — Installed PWA never picked up a new deploy on iPhone
**Status:** Resolved (2026-10-02)
**Description:** iOS Home Screen apps are usually *resumed* from a suspended state rather than reloaded when tapped open, so the service worker's normal "check for an update on navigation" path often never ran at all. Tom's own phone stayed on a build days old with no way to even tell which version he was looking at.
**Resolution:** The app now calls `registration.update()` every time it returns to the foreground (`visibilitychange`), on top of the existing autoUpdate/`skipWaiting` config, so a resumed session actively looks for a new service worker instead of waiting on a navigation that may never happen; Home shows a build-time "Updated MM/DD/YYYY h:mm AM/PM" stamp (America/Chicago) so it's visible at a glance which deploy is live on a given phone (TP-068).

### ISS-033 — History required tapping "Mine" to see a player clicked from Rankings
**Status:** Resolved (2026-10-02)
**Description:** Tapping a player from Rankings/Players opened `/history/:playerId` landed on "All players" by default rather than that player's own stats view, because a `useEffect` reset the tab to `'mine'`/`'all'` based on route changes in a way that didn't distinguish "just arrived at a specific player's route" from "player toggled tabs."
**Resolution:** `/history/:playerId` now always opens directly on that player's detail view; the first tab is relabeled with their short name when it isn't you ("Mine" otherwise), with a "View my history" link to get back to your own. Also used as the base for the staff-visibility work (TP-063): staff, who have no "mine," skip the detail tab entirely at a bare `/history` and open straight to "All players," but still see a normal detail view at `/history/:playerId`.

### ISS-034 — iOS numeric keypad had no minus sign for longitude
**Status:** Resolved (2026-10-02)
**Description:** Settings' Weather location Latitude/Longitude inputs used `inputMode="decimal"`, whose iOS keypad has no minus key — making a negative longitude (every longitude in North America) impossible to type on an iPhone.
**Resolution:** Changed to `type="text"`/`inputMode="text"`, which restores the full keyboard (including minus and period) while staying a plain text field; parsing and the existing paste-a-"lat, lng"-pair behavior were left unchanged. Range validation exists only server-side in `updateLeagueSettings`, unaffected by this change.

### ISS-035 — Admin Roster select mode could text selected players but not invite them
**Status:** Resolved (2026-10-02)
**Description:** The select-mode action bar on Admin Roster had "Text selected" but no equivalent bulk invite action, even though `sendInvites({playerIds})` already supported it.
**Resolution:** Added "Invite selected" next to "Text selected," reusing the exact existing `sendInvites` call shape; skips players already signed in, inactive, or who declined, with an inline (non-`window.confirm`) confirmation showing the skip-reason counts before sending. No select-all control exists yet. "Invite all" (TP-039) stays held — see TP-069 for why Tom is using "Invite selected" for the Oct 5 send instead.

### ISS-036 — Home's dinner/golf-sim card matched the signed-in player by display name
**Status:** Resolved (2026-10-02)
**Description:** `myDinnerBucket`/`myGolfSimBucket` on Home found "my" time bucket by checking whether the bucket's `names` array included the signed-in player's display name string — two players sharing an identical full name could collide and pick the wrong bucket, or wrongly exclude/include someone from "Also at dinner around …".
**Resolution:** `src/lib/socialTimes.js`'s buckets now also carry a `playerIds` array alongside `names`; Home matches by `playerId` instead, and `othersAtMyDinnerTime` was fixed the same way. `Matches.jsx`, the only other reader of these buckets, only ever used `time`/`names` and needed no change.

### ISS-037 — `notifyAutomationCrash`'s hourly cooldown compared against the wrong clock
**Status:** Resolved (2026-10-02)
**Description:** `notifyAutomationCrash(db, error, nowMs)` takes an explicit `nowMs` specifically so it can be driven deterministically (by `runWatchdog`'s own pattern, and tests), but wrote `lastCrashAlertAt: FieldValue.serverTimestamp()` — the real wall clock — instead of deriving the stored value from `nowMs`. Caught by a tests-only prompt that refused to bend its own test to match the implementation (per the stated rule for that prompt) rather than quietly working around it; in production this mostly self-corrected since callers normally pass `Date.now()`, but it was untestable with simulated time and could drift under retries.
**Resolution:** Now writes `Timestamp.fromMillis(nowMs)`; the read side needed no change, since it already calls `.toMillis()` on whatever's stored. Redeployed `weeklyAutomation` and `automationWatchdog`.

### ISS-038 — Decline link recorded a decline on GET, so email scanners declined players
**Status:** Resolved (2026-10-03)
**Description:** Within minutes of the Oct 3 invite send, 9 players showed as Declined under Roster → Not Active: Matt Fahey, Dave Frieder, Sebastien Imbert, Pat McDonough, Ben Pavlik, Jason Pickart, Matt Rinka, Tony Sarnowski and Steve Sewart. All 9 were at work addresses, and one confirmed he never clicked anything. The `decline` HTTP handler wrote `inviteStatus: 'declined'` and `declinedAt` on a plain GET, so corporate link scanners triggered it.
**Resolution:** A GET (and HEAD/OPTIONS) now only renders a confirm page; only its own POST writes (TP-070). Verified live after deploy: public access carried over, and a GET returns the confirm page without writing. Commit `3afed58`; `functions:decline` deployed. Note: the re-invites under TP-071 cleared `declinedAt` for these 9, so Firestore no longer shows the decline-vs-invite timing for them. Any trace of it may survive only in function logs.

### ISS-039 — "Resend invite" silently did nothing for a declined player
**Status:** Resolved (2026-10-03)
**Description:** Tom changed Ben Pavlik's email, saved, and tapped "Resend invite." Ben stayed Declined and no email went out. `resolveInviteTargets` dropped every `declined` player, including an admin's explicit `playerIds` pick.
**Resolution:** The explicit-`playerIds` path now includes a declined player who has no `firstSignInAt` and isn't `accepted` (TP-071). Ruled out first: `upsertPlayer` never touches `inviteStatus` on an edit, and the button calls `sendInvites({ playerIds })` with no client-side filter. Commit `f197bcc`; `functions:sendInvites` deployed. Tom re-sent to all 9 from their edit sheets, and all 9 moved to Pending immediately.

### ISS-040 — "Resend invite" gave no visible result
**Status:** Resolved (2026-10-03)
**Description:** The button showed "Sending…" and then reverted to its label, with no success or failure shown, so Tom couldn't tell whether the invite went out.
**Resolution:** `PlayerEditSheet.jsx` shows an inline result in the existing `.notice` styles (TP-072): green on success, red on failure or a skipped player, with the reason for a skip. Commit `685e257`; built and hosting deployed. Tom confirmed the new Updated stamp on his phone.
