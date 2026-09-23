# TNPL — Issues Tracker

**Last updated:** 2026-09-23

## Open

### ISS-001 — Two-match volunteers: back-to-back slots or a gap?
**Status:** Open
**Description:** For `canPlayTwo` players who play twice, the engine doesn't prefer adjacent slots or a gap between matches. The seed run produced both (6:00 + 7:15 and 6:00 + 8:30). Needs a decision from Tom before it becomes an engine rule.

### ISS-002 — TNPL missing from MASTER_CLAUDE_PROTOCOL.md tables
**Status:** Open
**Description:** TNPL isn't in the Active Projects table (Section 2) and prefix `TP` isn't in the DecisionLog prefix table (Section 11) of the root `MASTER_CLAUDE_PROTOCOL.md`. Root file, so left unchanged pending Tom's call (the Notion page link for the Section 2 row is also needed).

### ISS-003 — Rules and functions not deployed
**Status:** Open
**Description:** `firestore.rules` and the `generatePairings` callable exist only locally / in the emulator. First deploy still to do; after the first functions deploy, public invoker access must be set manually in the Cloud Run console.

### ISS-004 — Mid-season opt-out unresolved
**Status:** Open
**Description:** See TP-015 in DecisionLog.md. The engine only reads `optedIn` at run time; publish and change requests will need the decision.

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
