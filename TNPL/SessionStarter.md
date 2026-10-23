# TNPL — Session Starter
**Protocol:** Load MASTER_CLAUDE_PROTOCOL.md before this file.
Repo: github.com/Whit19/dataforge-standards

**Project:** Thursday Night Paddle League (TNPL) PWA
**Status:** Live and in use. Weeks 1 and 2 played. Tom's 7 post-Week-2 notes were all built and deployed Fri Oct 23: Rankings shows every enrolled player with a career Provisional rule, Info is a hub with Rules / Elo / Menu & Golf Sim / Music, Home shows your season and last week's results after a lock, a per-set match details page, and a new weekly flow where locking a week opens the next one right away with a results + "Are you in?" email.
**Last updated:** 2026-10-23

## One-line description
A PWA for Thursday Night Paddle League: live scoring, member sign-up, Elo rankings, and weekly match creation/viewing — similar to the UP Golf and Club Golf PWAs.

## Current status
Deployed and in real use. Firestore rules, Storage rules, all 44 Cloud Functions, and hosting are live on `tnpl-pwa`. Test suite: **568 passing** (`npm test` from the repo root); the weekly-loop rehearsal is **112/112** (rewritten for the new flow, 5 simulated weeks incl. a session break).

**Fri Oct 23 — post-Week-2 round, in 4 deploys + 1 follow-up + 1 one-time send:**
- **Rankings (TP-081):** every enrolled player in the active season, "Not played yet" rows mixed in by starting Elo; Provisional = under 12 sets **counting past seasons**; "1 match" fix. (`39df161`)
- **Info hub (TP-082):** Rules and Elo public, Menu & Golf Sim and Music sign-in only; Music in a members-only `leagueInfo/members` doc; editor "Shows on" picker + "Add starter Elo text". Rules + hosting deployed. (`0106d6e`)
- **Home after lock + match details (TP-083, TP-084):** "Your season" card and last week's results with an Elo column; `/matches/:id/details` with per-set Elo; details links on Matches. (`4b55520`)
- **Weekly flow (TP-085–TP-087):** Monday 6 AM auto-lock; any lock flags the next week, and the 15-minute automation opens it and sends the results + Are-you-in email; Monday nudge to non-responders; no early open across a session break; admin copies of every player email; full-schedule emails to staff + admins. Rehearsal 112/112 before deploy. (`f93f12f`)
- **Same-morning nudge rule (TP-088):** no Monday nudge when the week opened that same Chicago morning (the auto-lock path). (`cdc9743`)
- **Week 2's missed results email (TP-089):** Week 2 was locked before the new flow, so `functions/scripts/sendWeekResults.js` replayed `afterWeekLocked`. Tom previewed, then ran `--apply` at 4:34 PM; the 4:48 PM run opened **Week 3 early** and sent the results + "Are you in for Oct 29?" email to all 34 Week 3 recipients. (`d2d232b`)

## Key dates
- **Week 3 (`weeks/2026-10-29`) is open now** (opened early Fri Oct 23). Mon Oct 26 8 AM: nudge to non-responders. Tue Oct 27: reminder. Wed Oct 28 9 AM: answers close and the draft builds.
- **Thu Oct 29: first live run of the new flow.** Lock Week 3 Thursday night → the results email for Week 3 and Week 4's opening should land within ~15 minutes. If it isn't locked by Mon Nov 2 6:00 AM and every set is saved, it auto-locks.
- **Deploy freeze is recurring**, every Wednesday morning through Thursday night of a league week. Next window: Wed Oct 28 morning through Thu Oct 29 night.
- **Before Thu Nov 19:** decide Thanksgiving's early-open behavior (TPI-051).
- **Before ~Dec 2, 2026:** re-date the weekly-loop rehearsal (TPI-052).

## Next priorities
1. Watch the Oct 29 lock: results email within 15 minutes, Week 4 open, then the Monday 8 AM nudge only to non-responders. Check the Functions logs for `open_availability` / `send_results`.
2. Tom, in Admin › Info edit: set "League setup" to **Top of Info** (and update its availability line), review the starter Elo text, add the real Music content (WiFi, Sonos, amp reset, door code), Save.
3. TPI-051 (Thanksgiving early open), decide before Nov 19.
4. TPI-050: show the Monday auto-lock / open-on-lock in Settings' Automatic messages panel.
5. Respect the recurring Wed–Thu deploy freeze; run the rehearsal before any functions deploy touching automation, availability, pairing, change requests or lock/Elo.
6. Batch D, remaining: push notifications, an admin progress bar on Home, custom profile fields.
7. TPI-041, TPI-042, TPI-043 (unchanged, low priority).
8. Season close: carry-over into the next season, and the "next season starting Elos" link on Season summary.
9. Whole-week cancel **before** pairings are sent: still reserved, not built.
10. Re-run `backfillDirectory.js` whenever a new season is created.
11. Standing habits: use "Link to {name}" (not plain Approve) on join requests from people already on the roster; record "not this season" replies via the edit sheet's "This season" section — there is no admin path to mark anyone declined.

## Key decisions so far
See DecisionLog.md — TP-001 through TP-089. The most recent entries:
- **TP-081:** Rankings lists every enrolled player; Provisional counts past seasons.
- **TP-082:** Info hub + sub-pages; members-only `leagueInfo/members` for Music.
- **TP-083 / TP-084:** Home after lock; match details page with per-set Elo.
- **TP-085:** Monday 6 AM auto-lock, open-on-lock, Monday nudge, session-break exception (supersedes the timing parts of TP-044 / TP-047).
- **TP-086 / TP-087:** the results + Are-you-in email; who gets which email (admin copies, staff schedule only).
- **TP-088:** no Monday nudge when the week opened that same morning.
- **TP-089:** Week 2's results sent after the fact via a one-time script.

## Open questions
- **Thanksgiving:** should a Nov 19 lock open Dec 3 ~two weeks early (current behavior), or should Dec 3 wait for its normal Monday? (TPI-051)
- **Should a re-invite keep the previous decline as history?** Currently a re-invite clears `declinedAt` (TP-071). Option: keep it as `lastDeclinedAt`.
- **Should the reminder skip a recorded `none` choice?** (TPI-043) Not urgent.
