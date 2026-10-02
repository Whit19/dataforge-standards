# TNPL — Session Starter

**Project:** Thursday Night Paddle League (TNPL) PWA
**Status:** In progress — the full weekly loop, weather, cancel-a-night/slot, and the automation watchdog are all complete and deployed; a first real tester pass found and fixed several bugs; remaining invites go out **Oct 5**
**Last updated:** 2026-10-02

## One-line description
A PWA for Thursday Night Paddle League: live scoring, member sign-up, Elo rankings, and weekly match creation/viewing — similar to the UP Golf and Club Golf PWAs.

## Current status
Deployed and in real use. Firestore rules, Storage rules, all 42 Cloud Functions, and hosting are live on `tnpl-pwa`. Going into this session: the full weekly loop (dress-rehearsed end to end in the emulator, TP-059), weather (Batch C, TP-060), and real contact privacy were all complete. This session added two full features and then fixed a round of bugs a real tester found on the live app:
- **Cancel a night or slot** (TP-061, TP-062): admin can cancel one or more slots after pairings are sent, with emails, a Home/Matches banner, Lock Week/Elo correctly skipping anyone who ends up with zero sets played, and pairing history excluding cancelled groups. Whole-week cancel before pairings are sent is reserved (`WEEK_STATUS.CANCELLED`) but not built.
- **Automation watchdog** (TP-066): an hourly check that emails admins if any weekly-automation step is more than 30 minutes overdue, plus a separate crash alert (with a 1-hour cooldown) if `weeklyAutomation` itself throws. A test caught the cooldown comparing against the real wall clock instead of the function's own injectable clock (ISS-037) — fixed to use `Timestamp.fromMillis(nowMs)` consistently.
- **Tester-found fixes:** a real player hit `permission-denied` on both Rankings' pre-season view (ISS-028) and History on an all-draft active season (ISS-029) — both were Firestore's list-query provability rule rejecting a query it couldn't statically prove was safe, not an actual access problem; both fixed by querying data already readable by the caller instead of widening any rule. Tracing that also surfaced `useWeekMatches` silently hiding every `permission-denied`, not just the one expected case (ISS-030) — narrowed to just that case. Separately: staff can now see Rankings/History/Season summary (TP-063, reverses TP-016); History opens directly on a clicked player instead of requiring a "Mine" tap first (ISS-033); an installed iPhone PWA wasn't picking up new deploys after being resumed from the background, fixed with a `visibilitychange`-triggered update check plus a visible build-time "Updated" stamp on Home (ISS-032/TP-068); iOS's numeric keypad had no minus sign for entering a longitude (ISS-034); Roster's select mode gained "Invite selected" (ISS-035/TP-069), used for the Oct 5 invite send instead of un-holding "Invite all"; and a batch of wording/UI fixes (invite email mentions Chrome on Android, "partner with each player once," the golf-sim plan bolded like dinner's and both matched by `playerId` instead of display name (ISS-036), "View menu" moved into a header pill component).
- **Standing process change (TP-067):** Claude Code now commits and works directly on `main` going forward, after a prompt-by-prompt branching habit left a few small fixes briefly unmerged and one deploy going out from the wrong branch.

Test suite: **392 passing** (`npm test` from the repo root), up from 356 at the start of this session.

## Key dates
- Remaining invites go out **Oct 5** (via Roster's new "Invite selected," not "Invite all") — a few were already sent earlier; this sends the rest in one combined batch.
- Follow-up window for anyone who hasn't responded: **Oct 5–11**.
- First automated availability email: **Mon Oct 12, 9 AM** — also the first day weather shows on Home.
- First pairings: **Wed Oct 14** (draft 9 AM, sent 5 PM).
- **Deploy freeze Oct 14–15** around the first live pairing/match cycle — no non-critical deploys while the first real week is in flight.
- **First league night: Thu Oct 15.**
- First lock / auto-lock: before **Wed Oct 21, 9 AM**.

## Next priorities
1. Send the remaining invites **Oct 5** via "Invite selected"; watch for responses through **Oct 11**.
2. Tom is fixing the live `/info` page's own "partner each player" wording by hand (content edit, not code — `infoText.js`'s `STARTER_SECTIONS` is seed-only and isn't what's rendered there).
3. Confirm enough players have responded/opted in by **Oct 12** ahead of the first automated availability email.
4. Respect the **Oct 14–15** deploy freeze around the first live pairing/match/lock cycle.
5. Push notifications, an admin progress bar on Home, and custom profile fields (rest of Batch D — message wording is already done, ISS-036).
6. Season close: carry-over into next season (fields already stored, not yet consumed) and the "next season starting Elos" link on Season summary.
7. Whole-week cancel before pairings are sent — reserved (`WEEK_STATUS.CANCELLED`), not built.
8. Re-run `backfillDirectory.js` whenever a new season is created (everyone starts unlisted until they choose for that season).

## Key decisions so far
See DecisionLog.md — TP-001 through TP-069. This session added: per-slot cancel design (TP-061) and its Lock Week/Elo zero-sets-played fix (TP-062); staff can view Rankings/History/Season summary, reversing TP-016 (TP-063); Rankings' pre-season list sourced from `directory` instead of a `sessionEnrollment` query (TP-064); History's `matchGroups` query restricted to complete weeks only (TP-065); the automation watchdog's design — hourly overdue-step check plus a separately-cooldown'd crash alert (TP-066); Claude Code commits directly on `main` going forward (TP-067); PWA update-on-resume plus a visible build stamp (TP-068); "Invite selected" replaces un-holding "Invite all" for the remaining sends (TP-069). See the prior session's entries (TP-001–TP-060) for the original engine design, the weekly loop, contact privacy, and Batch C weather.

## Open questions
- Whether staff should see more/less than the read-only weekly schedule: largely settled this session (staff now also see Rankings/History/Season summary, TP-063) — still open whether staff need anything beyond read access anywhere else.
