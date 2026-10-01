# TNPL — Session Starter

**Project:** Thursday Night Paddle League (TNPL) PWA
**Status:** In progress — the full weekly loop (dress-rehearsed end to end in the emulator) and weather are now complete; invites still need to go out by Oct 4
**Last updated:** 2026-10-01

## One-line description
A PWA for Thursday Night Paddle League: live scoring, member sign-up, Elo rankings, and weekly match creation/viewing — similar to the UP Golf and Club Golf PWAs.

## Current status
Deployed and in real use. Firestore rules, Storage rules, all 39 Cloud Functions, and hosting are live on `tnpl-pwa`. The full weekly loop (availability → pairing → live scoring → Elo), change requests, Rankings/History/Season summary, admin Settings, Season 25-26 history import, week extras, and real contact privacy (see 2026-09-30 below) were all complete going into this session. This session: ran a full emulator dress rehearsal of the whole weekly loop end to end (`functions/scripts/runWeeklyLoop.js`, 74/74 PASS — TP-059), which found and fixed a real production bug (`directorySync` crashing on a `players` doc missing `seasonStartElo`, e.g. a change-request "replace with a guest" — TP-058/ISS-027, now writes `null` instead); wrote a plain-language Elo explainer for the `/info` page (content only, pasted in by Tom, no code); and built and deployed Batch C — a Home weather card (Open-Meteo, cached per-phone, shown Monday–Thursday of a play week, a coldest-slot layer badge, wind, rain/snow by forecast snowfall) with a new "Weather location" Settings row (TP-060). Test suite: 356 passing (`npm test` from the repo root), up from 340 at the start of this session. The weather card is built but not yet seen live — nothing shows on Home until Season 26-27's first Monday, **Oct 12**, regardless of when the venue is set.

## Key dates
- Invites go out by **Oct 4**, so the first automated Monday availability email (Oct 12) has opted-in players.
- First automated availability email: **Mon Oct 12, 9 AM**.
- First pairings: **Wed Oct 14** (draft 9 AM, sent 5 PM).
- **First league night: Thu Oct 15.**
- First lock / auto-lock: before **Wed Oct 21, 9 AM**.

## Next priorities
1. Set the real Weather location (Tripoli's coordinates) in Settings — the card stays hidden with none set.
2. Send invites by **Oct 4**, ahead of the first automated Monday availability email (Oct 12).
3. Optionally add `weatherVenue` tests to `functions/settings/settings.test.js` (no tests there yet for that field).
4. Message wording + custom profile fields (Batch D).
5. Season close: carry-over into next season (fields already stored, not yet consumed) and the "next season starting Elos" link on Season summary.
6. Push notifications.
7. Re-run `backfillDirectory.js` whenever a new season is created (everyone starts unlisted until they choose for that season) — and if a guest player is ever created before then, confirm its `playerRatings.seasonStartElo` is `null`, not missing (TP-058).

## Key decisions so far
See DecisionLog.md — TP-001 through TP-060. Notably beyond the original engine design (TP-004 through TP-023): mid-season opt-out raises a change request instead of silently dropping the player (TP-027); Season 26-27's dates and per-session choice (TP-029); weekly automation schedule (TP-030); emailed 6-digit code as the primary sign-in method (TP-031); email sends through Gmail, not Resend, for now (TP-032); invite email v2 is "install-first" (TP-035); the iPhone install gate (TP-036); the bottom tab bar and Home season card (TP-037); `firstSignInAt` is the durable "signed in" marker (TP-038); "Invite all" is deliberately held back (TP-039); the four-tab Roster redesign (TP-041); the public Rules & league info page (TP-042); change requests deferred for Week 1, then built (TP-043, superseded by TP-049/TP-050); weekly availability's one-tap email tokens (TP-044); any-order score entry (TP-045); Elo's same-week rule (TP-046); Lock Week's Tuesday-reminder/Wednesday-auto-lock/unlock policy (TP-047); lock/unlock run inside a Firestore transaction (TP-048); the change-request transaction/window model (TP-049/TP-050); Rankings/History computed on-device from `eloHistory` (TP-051); the Season 25-26 history import (TP-052); season-wide Settings and per-week time-slot overrides (TP-053); menu-as-PDF in Storage and locking `seasons`/`weeks` to server-write only (TP-054); week extras (TP-055); real contact privacy via `directory`/`playerRatings` (TP-056); the League contact replacing an `isAdmin` scan of `players`, and locking `players` reads to admin + self (TP-057); `directory`/`playerRatings` write `null`, never `undefined`, for a missing Elo field (TP-058); the weekly-loop dress rehearsal's clock/ordering approach (TP-059); Batch C's weather design — Open-Meteo, 60-minute per-phone cache, coldest-slot layer badge, snowfall-based rain/snow, no default venue coordinates (TP-060).

## Open questions
- Whether staff should see more/less than the read-only weekly schedule: working assumption stated in TechnicalArchitecture.md, not yet challenged by Tom.
