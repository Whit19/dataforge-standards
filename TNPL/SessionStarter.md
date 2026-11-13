# TNPL — Session Starter
**Protocol:** Load MASTER_CLAUDE_PROTOCOL.md before this file.
Repo: github.com/Whit19/dataforge-standards

**Project:** Thursday Night Paddle League (TNPL) PWA
**Status:** Live and in use. Weeks 1 through 5 have played (through Thu Nov 12) on the new auto-lock / open-on-lock flow shipped Oct 23. Two small bugs reported live by Tom since then were fixed same-day: the Availability tracker jumping to the wrong week (Oct 21), and a two-match player's results cards both showing the week's combined Elo instead of each match's own (Nov 13).
**Last updated:** 2026-11-13

## One-line description
A PWA for Thursday Night Paddle League: live scoring, member sign-up, Elo rankings, and weekly match creation/viewing — similar to the UP Golf and Club Golf PWAs.

## Current status
Deployed and in real use. Firestore rules, Storage rules, all 44 Cloud Functions, and hosting are live on `tnpl-pwa`. Test suite: **573 passing** (`npm test` from the repo root); the weekly-loop rehearsal is **112/112** (rewritten for the new flow, 5 simulated weeks incl. a session break) — unaffected by the two fixes below, neither touched `functions/`.

**Wed Oct 21 — Availability tracker fix (TP-091, TPI-053).** Tom clicked Admin → Availability on Week 2's draft-build day and landed on Week 3 instead. `defaultWeek()` only recognized `availability_open`/`draft`; `pairing_draft` added to the priority chain so a week stays the one shown right up until pairings send, not just until its draft is built. Also: the three count tiles are now the filter selector themselves, and the duplicate segmented control under them is gone. Hosting-only, deployed during the Wed–Thu freeze at Tom's explicit request (`5e6bf75`).

**Fri Nov 13 — two-match Elo display fix (TP-092, TPI-054).** Tom played two matches Thu Nov 12 (Week 5) and saw the same "+2 Elo" on both results cards. Home's results and Matches' Full schedule were both reading each player's single whole-week `eloHistory.delta` into a flat map and reusing it for every card — the correct per-match sum already existed (`matchEloBreakdown`, built for the match-details page, TP-084) and just wasn't being reused. New `groupEloByPlayer({ group, weekDocs })` alongside it; both pages now compute a fresh per-group map per card. Display-only, no Elo engine/lock change, no functions touched. 2 new tests (573 total); hosting-only deploy (`9224735`).

**Fri Oct 23 — post-Week-2 round, in 4 deploys + 1 follow-up + 1 one-time send:**
- **Rankings (TP-081):** every enrolled player in the active season, "Not played yet" rows mixed in by starting Elo; Provisional = under 12 sets **counting past seasons**; "1 match" fix. (`39df161`)
- **Info hub (TP-082):** Rules and Elo public, Menu & Golf Sim and Music sign-in only; Music in a members-only `leagueInfo/members` doc; editor "Shows on" picker + "Add starter Elo text". Rules + hosting deployed. (`0106d6e`)
- **Home after lock + match details (TP-083, TP-084):** "Your season" card and last week's results with an Elo column; `/matches/:id/details` with per-set Elo; details links on Matches. (`4b55520`)
- **Weekly flow (TP-085–TP-087):** Monday 6 AM auto-lock; any lock flags the next week, and the 15-minute automation opens it and sends the results + Are-you-in email; Monday nudge to non-responders; no early open across a session break; admin copies of every player email; full-schedule emails to staff + admins. Rehearsal 112/112 before deploy. (`f93f12f`)
- **Same-morning nudge rule (TP-088):** no Monday nudge when the week opened that same Chicago morning (the auto-lock path). (`cdc9743`)
- **No early open across a gap (TP-090, TPI-051 option 2):** only the very next Thursday opens early; Thanksgiving's Nov 19 lock sends results only and Dec 3 opens Mon Nov 30. Rehearsal 112/112, functions deployed. (`4ae8688`)
- **Week 2's missed results email (TP-089):** Week 2 was locked before the new flow, so `functions/scripts/sendWeekResults.js` replayed `afterWeekLocked`. Tom previewed, then ran `--apply` at 4:34 PM; the 4:48 PM run opened **Week 3 early** and sent the results + "Are you in for Oct 29?" email to all 34 Week 3 recipients. (`d2d232b`)

## Key dates
- **Weeks 1–5 have played and locked** (through Thu Nov 12) on the new flow — confirmed working end to end across 5 real weeks, not just the rehearsal.
- **Week 6 (`weeks/2026-11-19`) is open now**, confirmed in Firestore: opened early right after Week 5's lock (`openEarlyRequestedAt` 2026-11-13 ~8:44 AM Central), Tuesday reminder already sent. Draft builds Wed Nov 18, 9 AM; pairings send 5 PM; league night Thu Nov 19.
- **Thu Nov 19 is also the Thanksgiving-gap test (TP-090).** Session 1 skips Nov 26, so locking Week 6 should send results only (no early open) — Dec 3 opens on its normal Monday, Nov 30. Worth confirming this actually happens as designed when it's locked.
- **Deploy freeze is recurring**, every Wednesday morning through Thursday night of a league week. Next window: Wed Nov 18 morning through Thu Nov 19 night.
- **Before ~Dec 2, 2026:** re-date the weekly-loop rehearsal (TPI-052) — getting close.

## Next priorities
1. Confirm Thu Nov 19's lock actually triggers the Thanksgiving-gap behavior (TP-090): results-only, no early open, Dec 3 opens Mon Nov 30 instead.
2. Re-date the weekly-loop rehearsal before it expires (~Dec 2, TPI-052).
3. TPI-050: show the Monday auto-lock / open-on-lock in Settings' Automatic messages panel.
4. Respect the recurring Wed–Thu deploy freeze; run the rehearsal before any functions deploy touching automation, availability, pairing, change requests or lock/Elo.
5. Batch D, remaining: push notifications, an admin progress bar on Home, custom profile fields.
6. TPI-041, TPI-042, TPI-043 (unchanged, low priority).
7. Season close: carry-over into the next season, and the "next season starting Elos" link on Season summary.
8. Whole-week cancel **before** pairings are sent: still reserved, not built.
9. Re-run `backfillDirectory.js` whenever a new season is created.
10. Standing habits: use "Link to {name}" (not plain Approve) on join requests from people already on the roster; record "not this season" replies via the edit sheet's "This season" section — there is no admin path to mark anyone declined.

## Key decisions so far
See DecisionLog.md — TP-001 through TP-092. The most recent entries:
- **TP-083 / TP-084:** Home after lock; match details page with per-set Elo.
- **TP-085:** Monday 6 AM auto-lock, open-on-lock, Monday nudge, session-break exception (supersedes the timing parts of TP-044 / TP-047).
- **TP-086 / TP-087:** the results + Are-you-in email; who gets which email (admin copies, staff schedule only).
- **TP-088:** no Monday nudge when the week opened that same morning.
- **TP-089:** Week 2's results sent after the fact via a one-time script.
- **TP-090:** a week opens early only when it's the very next Thursday — a skipped week (Thanksgiving) or session break means results go out alone and the next week opens on its normal Monday.
- **TP-091:** Availability tracker stays on the relevant week through `pairing_draft`; count tiles are the filter themselves.
- **TP-092:** a two-match player's results cards each show that match's own Elo change, not the week's combined total on both.

## Open questions
- **Should a re-invite keep the previous decline as history?** Currently a re-invite clears `declinedAt` (TP-071). Option: keep it as `lastDeclinedAt`.
- **Should the reminder skip a recorded `none` choice?** (TPI-043) Not urgent.
