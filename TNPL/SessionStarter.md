# TNPL — Session Starter

**Project:** Thursday Night Paddle League (TNPL) PWA
**Status:** Live and in use. All league invites were sent on Sat Oct 3 (two days ahead of plan), and a launch-day triage fixed a scanner-caused false-decline problem. Next hard dates: follow-ups through Oct 11, the first automated email on Mon Oct 12, and the Oct 14–15 deploy freeze.
**Last updated:** 2026-10-03

## One-line description
A PWA for Thursday Night Paddle League: live scoring, member sign-up, Elo rankings, and weekly match creation/viewing — similar to the UP Golf and Club Golf PWAs.

## Current status
Deployed and in real use. Firestore rules, Storage rules, all 42 Cloud Functions, and hosting are live on `tnpl-pwa`. Test suite: **398 passing** (`npm test` from the repo root).

Going into Oct 3, the weekly loop, weather, cancel-a-night/slot, the automation watchdog, and the tester-found permission and PWA fixes were all complete (see DecisionLog TP-061 to TP-069 and IssuesTracker ISS-028 to ISS-037).

On Oct 3:
- **Invites sent early.** Tom sent the remaining invites with Roster's "Invite selected" (TP-069) instead of the planned Oct 5 send. Tom confirmed in Firestore that nothing else goes out automatically before Oct 12 (see Key dates).
- **Nine false declines, fixed.** Work-email security scanners opened the invite's decline link, and the link recorded the decline on page load. Nine players showed as Declined within minutes (ISS-038). The link now shows a confirm page and records a decline only on the button's POST (TP-070, commit `3afed58`, `functions:decline` deployed).
- **"Resend invite" skipped declined players.** A single-player re-invite now reaches a declined player who never signed in (TP-071, commit `f197bcc`, `functions:sendInvites` deployed). Bulk "Invite all"/"Invite selected" still skip decliners. Tom re-sent to all nine from their edit sheets, and all nine moved to Pending.
- **Resend invite shows its result.** The edit sheet now reports "Invite sent," "Not sent — {reason}," or an error inline (TP-072, ISS-040, commit `685e257`, hosting deployed). Tom confirmed the new build stamp on his phone.
- **Checked, not bugs:** a Roster list that seemed not to refresh was an older app bundle still on the phone, since `useRoster` is already a live listener (see BestMethods, iPhone PWA). A floating tab bar on the Admin dashboard can't be reproduced yet and is logged as ISS-042.

## Key dates
- **All league invites sent: Sat Oct 3.** Follow-up window for anyone who hasn't responded: **Oct 3–11**.
- **Automated Week 1 schedule** (Chicago time, from `weeks/2026-10-15`): availability email **Mon Oct 12, 9:00 AM**; reminder **Tue Oct 13, 7:00 PM**; draft built **Wed Oct 14, 9:00 AM**; pairings sent **Wed Oct 14, 5:00 PM**; finalized **Thu Oct 15, 8:00 AM**. Nothing is scheduled to go out automatically on Oct 5.
- **Deploy freeze Oct 14–15** around the first live pairing/match cycle — no non-critical deploys while the first real week is in flight.
- **First league night: Thu Oct 15.**
- First lock / auto-lock: before **Wed Oct 21, 9 AM**.

## Next priorities
1. Watch responses and follow up with anyone who hasn't answered through **Oct 11**.
2. Watch for new declines. A genuine decline is now a POST, so its `declinedAt` should not appear within seconds of `invitedAt`. A decline that does is worth a look.
3. Confirm enough players have opted in before the first automated email on **Oct 12**.
4. Tom is fixing the live `/info` page's own "partner each player" wording by hand. That's a content edit, not code: `infoText.js`'s `STARTER_SECTIONS` is seed-only and isn't what's rendered there.
5. Respect the **Oct 14–15** deploy freeze. Ship any front-end or function fix before Oct 14, or wait until after Oct 15.
6. Batch D, remaining: push notifications, an admin progress bar on Home, and custom profile fields. Message wording is already done (ISS-036).
7. After Oct 15: ISS-041 (the edit sheet's point-in-time player copy) and ISS-042 (tab bar floating on Admin, if it recurs).
8. Season close: carry-over into the next season (fields already stored, not yet consumed) and the "next season starting Elos" link on Season summary.
9. Whole-week cancel before pairings are sent: reserved (`WEEK_STATUS.CANCELLED`), not built.
10. Re-run `backfillDirectory.js` whenever a new season is created (everyone starts unlisted until they choose for that season).

## Key decisions so far
See DecisionLog.md — TP-001 through TP-072. The most recent entries:
- **TP-069:** "Invite selected" replaces un-holding "Invite all" for the remaining sends.
- **TP-070:** the decline link shows a confirm page on GET and records the decline only on POST, because email scanners open every link.
- **TP-071:** a single-player re-invite may reach a declined player who never signed in; bulk paths still skip decliners.
- **TP-072:** the edit sheet reports the real invite outcome inline, with a reason for skipped players.

Earlier entries (TP-001 to TP-068) cover the engine design, the weekly loop, cancel-a-night/slot, the staff visibility reversal, the automation watchdog, Claude Code committing on `main`, and the PWA update-on-resume and build stamp.

## Open questions
- **Should a re-invite keep the previous decline as history?** Currently a re-invite clears `declinedAt` (TP-071). That erased the decline-vs-invite timing for the nine scanner-caused declines in Firestore. Option: keep it as `lastDeclinedAt`.
- Whether staff should see more than the read-only schedule. Largely settled by TP-063 (staff can now view Rankings, History and Season summary). Still open whether staff need anything beyond read access.
