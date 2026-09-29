# TNPL — Session Starter

**Project:** Thursday Night Paddle League (TNPL) PWA
**Status:** In progress
**Last updated:** 2026-09-28

## One-line description
A PWA for Thursday Night Paddle League: live scoring, member sign-up, Elo rankings, and weekly match creation/viewing — similar to the UP Golf and Club Golf PWAs.

## Current status
Deployed and in real use. Firestore rules, all 17 Cloud Functions, and hosting are live on `tnpl-pwa`. Done: the full pairing engine plus publish/hold/finalize and 15-minute weekly automation; season setup (Season 26-27 created); admin dashboard and pairing-draft/edit screens; roster admin (add/edit players, invites, decline page, join requests); invite email v2 ("install-first," built from the real app icon); the iPhone install gate (Safari vs. the installed Home Screen app); player profile with season sign-up; and a bottom tab bar with a Home "Are you playing?" card. Tom's own self-invite and sign-in have been verified live. Test suite: 115 passing (`npm test` from the repo root). Not yet built: availability form, live scoring/score entry, Lock Week/Elo calc, Rankings/history, change-request admin approve/deny UI, and "Invite all" (deliberately held).

## Next priorities
1. **Roster tab redesign** — "Active" currently shows every never-invited player alongside signed-in players (ISS-014); needs something like Signed in / Invited / Not invited / Requests / Inactive.
2. Tom's iPhone check of the last two batches (6-tab layout at phone width, the install gate end to end).
3. Availability: the form itself, Monday-morning email (first one Oct 12), Tuesday reminder, admin response tracker.
4. Player Matches page (compact style) and score entry.
5. Lock Week, Elo calc, Rankings/history, Season 25-26 history import from the MASTER sheet.
6. Push notifications, hide-contact toggles, cancel a week/slot, weather, settings, rules page, admin progress bar on Home, a check that Cloud Scheduler automation actually ran.
7. "Invite all" — once the weekly loop above works end to end.

## Key decisions so far
See DecisionLog.md — TP-001 through TP-040. Notably beyond the original engine design (TP-004 through TP-023): mid-season opt-out raises a change request instead of silently dropping the player (TP-027); Season 26-27's dates and per-session choice (TP-029); weekly automation schedule (TP-030); emailed 6-digit code as the primary sign-in method, because iPhone links open Safari, not the installed app (TP-031); email sends through Gmail, not Resend, for now (TP-032); invite email v2 is "install-first" (TP-035); the iPhone install gate (TP-036); the bottom tab bar and Home season card (TP-037); `firstSignInAt` (not `inviteStatus`) is the durable "signed in" marker (TP-038); "Invite all" is deliberately held back (TP-039).

## Open questions
- **ISS-014 (Roster tab redesign) is priority 1** — see IssuesTracker.md and ProjectRoadmap.md.
- Whether staff should see more/less than the read-only weekly schedule: working assumption stated in TechnicalArchitecture.md, not yet challenged by Tom.
