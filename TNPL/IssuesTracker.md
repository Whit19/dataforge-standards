# TNPL — Issues Tracker

**Last updated:** 2026-09-28

## Open

### ISS-002 — TNPL missing from MASTER_CLAUDE_PROTOCOL.md tables
**Status:** Open
**Description:** TNPL isn't in the Active Projects table (Section 2) and prefix `TP` isn't in the DecisionLog prefix table (Section 11) of the root `MASTER_CLAUDE_PROTOCOL.md`. Root file, so left unchanged pending Tom's call (the Notion page link for the Section 2 row is also needed).

## Deferred

### ISS-005 — `availability` docs lack `weekId`/`playerId` fields
**Status:** Deferred
**Description:** The callable finds a week's availability by document-ID prefix `{weekId}_` — a workaround. Plan: write `weekId` and `playerId` fields when the availability form is built (the seed script already does; both lookups work).

### ISS-006 — Per-slot pairing re-run on change-request approval
**Status:** Deferred
**Description:** Designed but not built; the callable re-runs the whole week only.

## Resolved

### ISS-007 — Preferred slot had no effect against unplayed slots
**Status:** Resolved (2026-09-23)
**Description:** Found in the emulator run: a preferred slot cost 0, tying with unplayed slots, so slot order decided (Allen Van Scoyk, history s1:0 s2:0 s3:2, preferred s3, landed in s1). A spec gap, not an implementation bug.
**Resolution:** Preferred slot now costs `PREFERRED_SLOT_BONUS = -1`; `assignSlots` pruning shifts per-group costs so the minimum is 0. A preference can still lose to groupmates' rotation cost — accepted (TP-023).

### ISS-008 — Original preferred-slot unit-test fixture couldn't tell a preference from none
**Status:** Resolved (2026-09-23)
**Description:** With only `slotTally {s3:5}`, the player's other slots cost 0 too, so the test proved nothing.
**Resolution:** Fixture gave the other slots small tallies; a new regression test covers the exact emulator case.

### ISS-009 — `TNPL_MAIN.xlsx` S3 ELO column held Season 2 final Elos
**Status:** Resolved (2026-09-23)
**Description:** The MAIN sheet's "S3 ELO" column held Season 2 FINAL Elos, not the regressed Next Season Start values.
**Resolution:** Tom corrected MAIN; verified to match Player List "Season 3 Start Elo" for all 36 Player List names. MAIN has 45 players, 10 with no Elo (`#N/A`): Steve Sewart, Chris Elkendier, Jon Biorkman, John Burkemper, Jon Gould, Nick Gustafson, John Mort, Dave Patzer, Adam Trafton, Dan Wycklendt.

### ISS-010 — Firestore emulator couldn't start (no Java)
**Status:** Resolved (2026-09-23)
**Description:** The emulator needs JDK 21+ (firebase-tools 15.30.2). Temurin 21 was installed via winget but its bin folder wasn't on PATH.
**Resolution:** Set User PATH and `JAVA_HOME` with `[Environment]::SetEnvironmentVariable` (not `setx`, which can truncate a long PATH), then fully restart VS Code — a new terminal tab isn't enough because it inherits VS Code's environment.

### ISS-001 — Two-match volunteers: back-to-back slots or a gap?
**Status:** Resolved (2026-09-28)
**Description:** For `canPlayTwo` players who play twice, the engine didn't prefer adjacent slots or a gap between matches. The seed run produced both (6:00 + 7:15 and 6:00 + 8:30).
**Resolution:** Engine keeps a soft preference for back-to-back slots (`TWO_MATCH_GAP_PENALTY = 2.1`), reviewed against a near-tie case in real data and kept as-is (TP-028).

### ISS-003 — Rules and functions not deployed
**Status:** Resolved (2026-09-28)
**Description:** `firestore.rules` and the `generatePairings` callable existed only locally / in the emulator.
**Resolution:** Deployed. `firestore.rules`, all 17 Cloud Functions, and hosting are live on `tnpl-pwa`; public invoker access was set in the Cloud Run console for each callable/HTTP function as it was added.

### ISS-004 — Mid-season opt-out unresolved
**Status:** Resolved (2026-09-28)
**Description:** See TP-015 in DecisionLog.md. The engine only read `optedIn` at run time; publish and change requests needed the decision.
**Resolution:** See TP-027 — opting out stops weekly emails, and raises a `swap_out` change request instead of silently removing the player if they're already in a published week's match.

### ISS-011 — `sendInvites` downgraded an already signed-in player back to "invited"
**Status:** Resolved (2026-09-28)
**Description:** Re-sending an invite (e.g. to a player who had already signed in) unconditionally overwrote `inviteStatus` to `'invited'` and reset `invitedAt`/the decline token, even though the player had already accepted. Roster then showed them under Invited instead of Active.
**Resolution:** `sendInvites` still sends the email but no longer touches `inviteStatus`/`invitedAt`/the decline token for anyone with `firstSignInAt` set or `inviteStatus: 'accepted'` (TP-038). A one-time backfill script set `firstSignInAt`/`inviteStatus` for the one player whose original link predated that field.

### ISS-012 — Invite-email Home Screen icon didn't match the real app icon
**Status:** Resolved (2026-09-28)
**Description:** The first version of the invite email's "here's what to look for" Home Screen image was a hand-drawn approximation that didn't match the actual installed icon.
**Resolution:** Regenerated the email images directly from the real `apple-touch-icon` file instead of redrawing them, so the email and the actual Home Screen icon match exactly.

### ISS-013 — Incorrect phone numbers from the roster workbook
**Status:** Resolved (2026-09-28)
**Description:** 12 players had incorrect phone numbers carried over from the original import workbook.
**Resolution:** Corrected via a one-time, dry-run-first script (names logged, numbers never logged) run directly against Firestore.

### ISS-015 — Google sign-in popup / Incognito testing quirks (not an app bug)
**Status:** Resolved (not a bug)
**Description:** Chrome can occasionally open the Google sign-in popup minimized or off-screen. Testing sign-in in an Incognito window is unreliable because Incognito blocks third-party cookies the popup flow depends on.
**Resolution:** Not an app defect — a browser/testing-environment quirk. Test sign-in in a normal (non-Incognito) window; if the Google popup seems to do nothing, check for a minimized window before assuming the flow is broken.

### ISS-014 — Roster "Active" tab lists every never-invited player
**Status:** Resolved (2026-09-28)
**Description:** With the three-tab `classify()` rule (TP-038), a brand-new admin-added player who hasn't been sent an invite yet counted as "Active" alongside players who have actually signed in, since "never invited" and "signed in" both landed in the same tab.
**Resolution:** Redesigned to four tabs — Active / Pending / To Invite / Not Active (TP-041). Active now requires both being signed in and having opted into a session; a signed-in player who chose "not this season" lands in Not Active with a note explaining why, distinct from an actual decline.
