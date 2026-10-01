# TNPL — Session Starter

**Project:** Thursday Night Paddle League (TNPL) PWA
**Status:** In progress — the full weekly loop, change requests, Rankings/History, Settings, and real contact privacy are now complete
**Last updated:** 2026-09-30

## One-line description
A PWA for Thursday Night Paddle League: live scoring, member sign-up, Elo rankings, and weekly match creation/viewing — similar to the UP Golf and Club Golf PWAs.

## Current status
Deployed and in real use. Firestore rules, Storage rules, all 39 Cloud Functions, and hosting are live on `tnpl-pwa`. Beyond the full weekly loop (availability → pairing → live scoring → Elo, complete as of 2026-09-29), this session added: change requests end to end (player request form, withdraw, admin `/admin/changes` approve/deny with the approval written inside the same transaction as the pairing edit); Rankings, History, and Season summary, computed on-device from `eloHistory`; admin Settings (Elo/pairing values, default + per-week time slots, skip a week, admins, golf-sim link, League contact, an uploaded menu PDF in Firebase Storage); Season 25-26 history import (17 weeks, 68 matches, verified against the Elo engine); week extras (a per-week dinner special and an optional yes/no question, with email/tracker support); and real contact privacy — a server-maintained `directory`/`playerRatings` model (listed only if active, signed in, and opted into the active season; staff once active), hide-phone/email switches, a League contact replacing "scan players for isAdmin," and the `players` collection finally locked to admin + self. Test suite: 340 passing (`npm test` from the repo root), up from 223 at the start of this session.

## Key dates
- Invites go out by **Oct 4**, so the first automated Monday availability email (Oct 12) has opted-in players.
- First automated availability email: **Mon Oct 12, 9 AM**.
- First pairings: **Wed Oct 14** (draft 9 AM, sent 5 PM).
- **First league night: Thu Oct 15.**
- First lock / auto-lock: before **Wed Oct 21, 9 AM**.

## Next priorities
1. A full emulator run-through of the whole weekly loop before real play resumes.
2. Weather (Batch C).
3. Message wording + custom profile fields (Batch D).
4. Season close: carry-over into next season (fields already stored, not yet consumed) and the "next season starting Elos" link on Season summary.
5. Push notifications.
6. Re-run `backfillDirectory.js` whenever a new season is created (everyone starts unlisted until they choose for that season).

## Key decisions so far
See DecisionLog.md — TP-001 through TP-057. Notably beyond the original engine design (TP-004 through TP-023): mid-season opt-out raises a change request instead of silently dropping the player (TP-027); Season 26-27's dates and per-session choice (TP-029); weekly automation schedule (TP-030); emailed 6-digit code as the primary sign-in method (TP-031); email sends through Gmail, not Resend, for now (TP-032); invite email v2 is "install-first" (TP-035); the iPhone install gate (TP-036); the bottom tab bar and Home season card (TP-037); `firstSignInAt` is the durable "signed in" marker (TP-038); "Invite all" is deliberately held back (TP-039); the four-tab Roster redesign (TP-041); the public Rules & league info page (TP-042); change requests deferred for Week 1, then built (TP-043, superseded by TP-049/TP-050); weekly availability's one-tap email tokens (TP-044); any-order score entry (TP-045); Elo's same-week rule (TP-046); Lock Week's Tuesday-reminder/Wednesday-auto-lock/unlock policy (TP-047); lock/unlock run inside a Firestore transaction (TP-048); the change-request transaction/window model (TP-049/TP-050); Rankings/History computed on-device from `eloHistory` (TP-051); the Season 25-26 history import (TP-052); season-wide Settings and per-week time-slot overrides (TP-053); menu-as-PDF in Storage and locking `seasons`/`weeks` to server-write only (TP-054); week extras (TP-055); real contact privacy via `directory`/`playerRatings` (TP-056); the League contact replacing an `isAdmin` scan of `players`, and locking `players` reads to admin + self (TP-057).

## Open questions
- Whether staff should see more/less than the read-only weekly schedule: working assumption stated in TechnicalArchitecture.md, not yet challenged by Tom.
