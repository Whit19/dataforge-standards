# TNPL — Session Starter
**Protocol:** Load MASTER_CLAUDE_PROTOCOL.md before this file.
Repo: github.com/Whit19/dataforge-standards

**Project:** Thursday Night Paddle League (TNPL) PWA
**Status:** Live and in use. Roster cleanup is done: a Pending-tab reminder email shipped, admins can set a player's season choice directly, duplicate players (from join requests) can be merged, a reminder-send timeout incident the same day was diagnosed and fixed, and admins can now directly set a player's availability for a week. Week 1 (Thu Oct 15) is in flight; the Oct 14–15 deploy freeze is active.
**Last updated:** 2026-10-14

## One-line description
A PWA for Thursday Night Paddle League: live scoring, member sign-up, Elo rankings, and weekly match creation/viewing — similar to the UP Golf and Club Golf PWAs.

## Current status
Deployed and in real use. Firestore rules, Storage rules, all 43 Cloud Functions, and hosting are live on `tnpl-pwa`. Test suite: **488 passing** (`npm test` from the repo root); the full weekly-loop rehearsal is still 74/74.

**Wed Oct 14 (same day as Week 1's pairing send):** Tom needed to add a player to that night's matches after their own availability-answer window had already closed. Built and shipped same-day — admin can now tap a player's name in Admin → Availability and set/edit their availability directly, even after the window closes (TP-077, `saveAvailability` gained an optional admin-only `playerId`). Deployed despite the Oct 14–15 freeze, at Tom's explicit request; the weekly-loop rehearsal was run *after* the deploy rather than before (should have been before — no regression found, still 74/74). Tom set the player's availability, re-ran `generatePairings`, and confirmed they landed in that night's matches.

Going into Oct 5, invites were fully sent, the Oct 3 false-decline scanner problem was fixed, and Roster was otherwise stable (see DecisionLog TP-061 to TP-072 and IssuesTracker TPI-028 to TPI-040).

Spanning Sun Oct 4 evening through Mon Oct 5 midday (~3 hours total):
- **Pending-tab reminder email (TP-073).** `sendInvites` gained `mode: 'reminder'`: two variants ("the app's ready when you are" for not-signed-in players, "one quick step left" for signed-in players with no season pick), sent from Roster's Pending tab (Select all → Send reminder), with a "Send a test to me" option. Never touches `inviteStatus`/`invitedAt`/`declinedAt`/the decline token — only a `reminderSentAt` stamp. The reminder can't reuse a player's original decline link (only a salted hash of that token is ever stored), so the not-signed-in variant says "Just reply to this email and let me know" instead of a dead link.
- **Known, deliberately deferred mismatch (TPI-043):** the reminder targets any not-signed-in player regardless of a recorded season choice, both server- and client-side. Low risk — flagged, not fixed, can wait past Oct 15.
- **Admin season-choice override (TP-074).** `setSeasonSignup` takes an optional `playerId`; an admin can record Full/Session 1/Session 2/Not this season for any active player, even one who's never signed in (confirmed: the Monday email and pairing eligibility key only on `optedIn`/`role`/`active`, never `firstSignInAt`). `classify()` now honors a recorded choice for a not-signed-in player too — previously ignored it outright. The edit sheet's new "This season" section is where Tom records a reply like "not this season"; there is still no admin path to mark anyone declined, and docs were corrected after chat initially implied otherwise.
- **Duplicate players, root cause fixed (TPI-044, TP-075).** Someone signing in with a different email than the roster one hit `not_on_roster`, and approving their join request always created a second player doc. New admin callable `mergeDuplicatePlayer` (refuses on any real game history; one transaction; repoints `playerLinks`, keeps Elo, carries over the season choice) merges a pair into one. `resolveJoinRequest` separately gained `linkToPlayerId` to route a *new* request onto an existing player before a duplicate is ever created. Tom merged the real "Brian Spahn" duplicate after setting Cloud Run access for the new function.
- **Verified without a browser.** Both features were checked by running the real server code against a Firestore emulator, since this environment has none.
- **Reminder-send timeout incident, diagnosed then fixed (TPI-045, TP-076).** Mon Oct 5 ~13:56 Central, Tom tapped "Send to 55" and saw a red `internal [0]` banner. A read-only diagnosis first (per Tom's explicit instruction) found `sendInvites` had no `timeoutSeconds`, so it hit the Functions v2 default of 60s; Cloud Run returned a 504 the client SDK couldn't parse, but the container kept running in the background and finished the whole batch — all 55 players got exactly one email, confirmed from `reminderSentAt` stamps, zero failures. Fix shipped in 3 ordered prompts: server `timeoutSeconds: 540` plus a 24h `reminded_recently` duplicate-send guard (server + client, `force` bypass server-only); then Roster's UI — a dedicated client `httpsCallable` with a matching 540000ms timeout, a friendly "may still finish" message instead of the raw error, "Reminded {date}" on Pending rows, and skip-count/button wording changes. Deployed (confirmed the live timeout via `gcloud run services describe`); hosting deployed. Test suite 472 → 483.

## Key dates
- **All league invites sent: Sat Oct 3.** Follow-up window through Oct 11 is done.
- **Week 1 is in flight right now** (Chicago time, `weeks/2026-10-15`): availability opened Mon Oct 12; reminder sent Tue Oct 13; draft built Wed Oct 14, 9:00 AM; pairings sent Wed Oct 14, 5:00 PM; finalize is Thu Oct 15, 8:00 AM.
- **Deploy freeze Oct 14–15** around the first live pairing/match cycle — one exception made same-day for TP-077 at Tom's explicit, urgent request (see Current status above); otherwise still in force through Thu night.
- **First league night: Thu Oct 15** (this doc was last updated Wed Oct 14, the day before).
- First lock / auto-lock: before **Wed Oct 21, 9 AM**.
- **Merges are only possible before a duplicate has any game history** (`eloHistory`/`matchGroups`/`availability`/`changeRequests`) — the Oct 14 draft has now been built, so any remaining merge needs to happen before that player plays.

## Next priorities
1. Get through Thursday Oct 15's first league night — watch for anything unexpected in live scoring/Lock Week, since this is the first real end-to-end run of the whole weekly loop.
2. Respect the **Oct 14–15** deploy freeze for anything other than a genuinely urgent, Tom-approved exception (see TP-077).
3. First lock / auto-lock before **Wed Oct 21, 9 AM** — this is the first time that path runs against real data.
4. Use "Link to {name}" on new join requests from people already on the roster, not plain Approve.
5. Record "not this season" replies through the edit sheet's "This season" section, not by trying to mark anyone declined (no such path exists).
6. Tom is fixing the live `/info` page's own "partner each player" wording by hand — a content edit, not code.
7. Batch D, remaining: push notifications, an admin progress bar on Home, and custom profile fields.
8. After the first week settles: TPI-041 (edit sheet point-in-time player copy), TPI-042 (tab bar floating on Admin, if it recurs), TPI-043 (reminder ignores a recorded choice for a not-signed-in player).
9. Season close: carry-over into the next season, and the "next season starting Elos" link on Season summary.
10. Whole-week cancel before pairings are sent: reserved (`WEEK_STATUS.CANCELLED`), not built.
12. Re-run `backfillDirectory.js` whenever a new season is created.

## Key decisions so far
See DecisionLog.md — TP-001 through TP-077. The most recent entries:
- **TP-074:** admin season-choice override; `classify()` fixed to honor a not-signed-in player's recorded choice.
- **TP-075:** duplicate-player merge (`mergeDuplicatePlayer`) and join-request linking (`linkToPlayerId`).
- **TP-076:** reminder-send timeout fix — 540s server + client timeout, 24h `reminded_recently` duplicate-send guard, friendly timeout UI.
- **TP-077:** admin availability override — `saveAvailability` gained an optional `playerId` so an admin can set/edit a player's availability for a week after their own window closes.

Earlier entries (TP-001 to TP-072) cover the engine design, the weekly loop, cancel-a-night/slot, the staff visibility reversal, the automation watchdog, the Oct 3 decline-scanner fix, and Resend-invite's inline result.

## Open questions
- **Should a re-invite keep the previous decline as history?** Currently a re-invite clears `declinedAt` (TP-071). Option: keep it as `lastDeclinedAt`.
- **Should the reminder skip a recorded `none` choice?** (TPI-043) Not urgent — can wait past Oct 15.
- Whether staff should see more than the read-only schedule. Largely settled by TP-063. Still open whether staff need anything beyond read access.
