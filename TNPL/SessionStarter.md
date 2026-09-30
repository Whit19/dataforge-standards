# TNPL — Session Starter

**Project:** Thursday Night Paddle League (TNPL) PWA
**Status:** In progress — the full weekly loop is now complete
**Last updated:** 2026-09-29

## One-line description
A PWA for Thursday Night Paddle League: live scoring, member sign-up, Elo rankings, and weekly match creation/viewing — similar to the UP Golf and Club Golf PWAs.

## Current status
Deployed and in real use. Firestore rules, all 24 Cloud Functions, and hosting are live on `tnpl-pwa`. The full weekly loop — availability → pairing → live scoring → Elo — is now built end to end: weekly availability collection (Monday email + Tuesday reminder, one-tap answers, in-app form, admin tracker); the pairing engine plus publish/hold/finalize and 15-minute weekly automation; season setup (Season 26-27 created); admin dashboard, pairing-draft/edit screens, and a Scores-and-lock screen; roster admin with a four-tab Active/Pending/To Invite/Not Active view; a public Rules & league info page (`/info`) and a public join-request page (`/join`); invite email v2 ("install-first") with iPhone install steps updated for iOS 26 Safari's ••• button; the iPhone install gate; player profile with season sign-up; a bottom tab bar; the player Matches page with live, any-order score entry; and Lock Week / Elo (pure engine ported from the Excel formulas, transactional lock/unlock with safety checks, Tuesday reminder + Wednesday auto-lock). Test suite: 223 passing (`npm test` from the repo root), up from 116 at the start of this session. Not yet built: Rankings/history, the change-request admin approve/deny UI ("Text Tom" stands in for Week 1), Season 25-26 history import, and a Settings page.

## Key dates
- Invites go out by **Oct 4**, so the first automated Monday availability email (Oct 12) has opted-in players.
- First automated availability email: **Mon Oct 12, 9 AM**.
- First pairings: **Wed Oct 14** (draft 9 AM, sent 5 PM).
- **First league night: Thu Oct 15.**
- First lock / auto-lock: before **Wed Oct 21, 9 AM**.

## Next priorities
1. Tom's real-device check once Week 1 is actually live (availability, matches, score entry, lock all against real play).
2. Change-request player + admin approve/deny UI (TP-006, deferred for Week 1 by TP-043).
3. Rankings / season history page.
4. Season 25-26 history import from the MASTER sheet.
5. Settings page (weekly special, menu, golf-sim booking link, custom availability question, editable Elo/pairing variables) — several already-built pieces are wired to appear once it exists.
6. Weather, push notifications, Players page.

## Key decisions so far
See DecisionLog.md — TP-001 through TP-048. Notably beyond the original engine design (TP-004 through TP-023): mid-season opt-out raises a change request instead of silently dropping the player (TP-027); Season 26-27's dates and per-session choice (TP-029); weekly automation schedule (TP-030); emailed 6-digit code as the primary sign-in method (TP-031); email sends through Gmail, not Resend, for now (TP-032); invite email v2 is "install-first" (TP-035); the iPhone install gate (TP-036); the bottom tab bar and Home season card (TP-037); `firstSignInAt` is the durable "signed in" marker (TP-038); "Invite all" is deliberately held back (TP-039); the four-tab Roster redesign (TP-041); the public Rules & league info page (TP-042); change requests deferred for Week 1 (TP-043); weekly availability's one-tap email tokens (TP-044); any-order score entry (TP-045); Elo's same-week rule (TP-046); Lock Week's Tuesday-reminder/Wednesday-auto-lock/unlock policy (TP-047); lock/unlock run inside a Firestore transaction (TP-048).

## Open questions
- Whether staff should see more/less than the read-only weekly schedule: working assumption stated in TechnicalArchitecture.md, not yet challenged by Tom.
