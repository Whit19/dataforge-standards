# TNPL — Session Starter

**Project:** Thursday Night Paddle League (TNPL) PWA
**Status:** In progress
**Last updated:** 2026-09-28

## One-line description
A PWA for Thursday Night Paddle League: live scoring, member sign-up, Elo rankings, and weekly match creation/viewing — similar to the UP Golf and Club Golf PWAs.

## Current status
Deployed and in real use. Firestore rules, all 17 Cloud Functions, and hosting are live on `tnpl-pwa`. Done: the full pairing engine plus publish/hold/finalize and 15-minute weekly automation; season setup (Season 26-27 created); admin dashboard and pairing-draft/edit screens; roster admin (add/edit players, invites, decline page, join requests) with a four-tab Active/Pending/To Invite/Not Active status view; invite email v2 ("install-first," built from the real app icon); the iPhone install gate (Safari vs. the installed Home Screen app); player profile with season sign-up; and a bottom tab bar with a Home "Are you playing?" card. Tom's own self-invite and sign-in have been verified live. Test suite: 116 passing (`npm test` from the repo root). Not yet built: availability form, live scoring/score entry, Lock Week/Elo calc, Rankings/history, change-request admin approve/deny UI, and "Invite all" (deliberately held).

## Next priorities
1. Tom's iPhone check of the last few batches (6-tab layout at phone width, the install gate end to end, the new Roster tabs).
2. Availability: the form itself, Monday-morning email (first one Oct 12), Tuesday reminder, admin response tracker.
3. Player Matches page (compact style) and score entry.
4. Lock Week, Elo calc, Rankings/history, Season 25-26 history import from the MASTER sheet.
5. Push notifications, hide-contact toggles, cancel a week/slot, weather, settings, rules page, admin progress bar on Home, a check that Cloud Scheduler automation actually ran.
6. "Invite all" — once the weekly loop above works end to end.

## Key decisions so far
See DecisionLog.md — TP-001 through TP-041. Notably beyond the original engine design (TP-004 through TP-023): mid-season opt-out raises a change request instead of silently dropping the player (TP-027); Season 26-27's dates and per-session choice (TP-029); weekly automation schedule (TP-030); emailed 6-digit code as the primary sign-in method, because iPhone links open Safari, not the installed app (TP-031); email sends through Gmail, not Resend, for now (TP-032); invite email v2 is "install-first" (TP-035); the iPhone install gate (TP-036); the bottom tab bar and Home season card (TP-037); `firstSignInAt` (not `inviteStatus`) is the durable "signed in" marker (TP-038); "Invite all" is deliberately held back (TP-039); the four-tab Roster redesign, Active/Pending/To Invite/Not Active (TP-041).

## Open questions
- Whether staff should see more/less than the read-only weekly schedule: working assumption stated in TechnicalArchitecture.md, not yet challenged by Tom.
