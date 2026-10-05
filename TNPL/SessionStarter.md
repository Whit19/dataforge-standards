# TNPL — Session Starter

**Project:** Thursday Night Paddle League (TNPL) PWA
**Status:** Live and in use. Roster cleanup is done: a Pending-tab reminder email shipped, admins can set a player's season choice directly, and duplicate players (from join requests) can be merged. Next hard dates: the Oct 12 availability email and the Oct 14–15 deploy freeze.
**Last updated:** 2026-10-05

## One-line description
A PWA for Thursday Night Paddle League: live scoring, member sign-up, Elo rankings, and weekly match creation/viewing — similar to the UP Golf and Club Golf PWAs.

## Current status
Deployed and in real use. Firestore rules, Storage rules, all 43 Cloud Functions, and hosting are live on `tnpl-pwa`. Test suite: **472 passing** (`npm test` from the repo root); the full weekly-loop rehearsal is still 74/74.

Going into Oct 5, invites were fully sent, the Oct 3 false-decline scanner problem was fixed, and Roster was otherwise stable (see DecisionLog TP-061 to TP-072 and IssuesTracker ISS-028 to ISS-040).

Spanning Sun Oct 4 evening through Mon Oct 5 midday (~3 hours total):
- **Pending-tab reminder email (TP-073).** `sendInvites` gained `mode: 'reminder'`: two variants ("the app's ready when you are" for not-signed-in players, "one quick step left" for signed-in players with no season pick), sent from Roster's Pending tab (Select all → Send reminder), with a "Send a test to me" option. Never touches `inviteStatus`/`invitedAt`/`declinedAt`/the decline token — only a `reminderSentAt` stamp. The reminder can't reuse a player's original decline link (only a salted hash of that token is ever stored), so the not-signed-in variant says "Just reply to this email and let me know" instead of a dead link.
- **Known, deliberately deferred mismatch (ISS-043):** the reminder targets any not-signed-in player regardless of a recorded season choice, both server- and client-side. Low risk — flagged, not fixed, can wait past Oct 15.
- **Admin season-choice override (TP-074).** `setSeasonSignup` takes an optional `playerId`; an admin can record Full/Session 1/Session 2/Not this season for any active player, even one who's never signed in (confirmed: the Monday email and pairing eligibility key only on `optedIn`/`role`/`active`, never `firstSignInAt`). `classify()` now honors a recorded choice for a not-signed-in player too — previously ignored it outright. The edit sheet's new "This season" section is where Tom records a reply like "not this season"; there is still no admin path to mark anyone declined, and docs were corrected after chat initially implied otherwise.
- **Duplicate players, root cause fixed (ISS-044, TP-075).** Someone signing in with a different email than the roster one hit `not_on_roster`, and approving their join request always created a second player doc. New admin callable `mergeDuplicatePlayer` (refuses on any real game history; one transaction; repoints `playerLinks`, keeps Elo, carries over the season choice) merges a pair into one. `resolveJoinRequest` separately gained `linkToPlayerId` to route a *new* request onto an existing player before a duplicate is ever created. Tom merged the real "Brian Spahn" duplicate after setting Cloud Run access for the new function.
- **Verified without a browser.** Both features were checked by running the real server code against a Firestore emulator, since this environment has none.

## Key dates
- **All league invites sent: Sat Oct 3.** Follow-up window through **Oct 11** is ongoing.
- **Automated Week 1 schedule** (Chicago time, from `weeks/2026-10-15`): availability email **Mon Oct 12, 9:00 AM**; reminder **Tue Oct 13, 7:00 PM**; draft built **Wed Oct 14, 9:00 AM**; pairings sent **Wed Oct 14, 5:00 PM**; finalized **Thu Oct 15, 8:00 AM**.
- **Deploy freeze Oct 14–15** around the first live pairing/match cycle — no non-critical deploys while the first real week is in flight.
- **First league night: Thu Oct 15.**
- First lock / auto-lock: before **Wed Oct 21, 9 AM**.
- **Merges are only possible before a duplicate has any game history** (`eloHistory`/`matchGroups`/`availability`/`changeRequests`) — finish any remaining merges before the Oct 14 draft, not after.

## Next priorities
1. Send the Pending-tab reminder (test first with "Send a test to me"), then watch responses through **Oct 11**.
2. Merge any other duplicate players before the Oct 14 draft — once a duplicate has played anything, `mergeDuplicatePlayer` refuses.
3. Use "Link to {name}" on new join requests from people already on the roster, not plain Approve.
4. Record "not this season" replies through the edit sheet's "This season" section, not by trying to mark anyone declined (no such path exists).
5. Confirm enough players have opted in before the first automated email on **Oct 12**.
6. Tom is fixing the live `/info` page's own "partner each player" wording by hand — a content edit, not code.
7. Respect the **Oct 14–15** deploy freeze.
8. Batch D, remaining: push notifications, an admin progress bar on Home, and custom profile fields.
9. After Oct 15: ISS-041 (edit sheet point-in-time player copy), ISS-042 (tab bar floating on Admin, if it recurs), ISS-043 (reminder ignores a recorded choice for a not-signed-in player).
10. Season close: carry-over into the next season, and the "next season starting Elos" link on Season summary.
11. Whole-week cancel before pairings are sent: reserved (`WEEK_STATUS.CANCELLED`), not built.
12. Re-run `backfillDirectory.js` whenever a new season is created.

## Key decisions so far
See DecisionLog.md — TP-001 through TP-075. The most recent entries:
- **TP-073:** Pending-tab reminder email, `sendInvites` `mode: 'reminder'`; fixed same-day to use a reply line instead of a dead decline link.
- **TP-074:** admin season-choice override; `classify()` fixed to honor a not-signed-in player's recorded choice.
- **TP-075:** duplicate-player merge (`mergeDuplicatePlayer`) and join-request linking (`linkToPlayerId`).

Earlier entries (TP-001 to TP-072) cover the engine design, the weekly loop, cancel-a-night/slot, the staff visibility reversal, the automation watchdog, the Oct 3 decline-scanner fix, and Resend-invite's inline result.

## Open questions
- **Should a re-invite keep the previous decline as history?** Currently a re-invite clears `declinedAt` (TP-071). Option: keep it as `lastDeclinedAt`.
- **Should the reminder skip a recorded `none` choice?** (ISS-043) Not urgent — can wait past Oct 15.
- Whether staff should see more than the read-only schedule. Largely settled by TP-063. Still open whether staff need anything beyond read access.
