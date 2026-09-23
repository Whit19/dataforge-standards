# TNPL — Best Methods

**Last updated:** 2026-09-22

Hard-won lessons for this project. Read before writing any code.

## Firebase provisioning
- **A fresh Firebase project does not auto-provision Firestore.** `firebase-tools` has no command to create the database, only to interact with one that already exists (`firebase firestore:databases:list` returns "No databases found" until you create it manually via console → Firestore Database → Create database). Always verify with that command before writing/deploying rules, rather than assuming the console setup is done.
- **Auth provider status isn't checkable via CLI/API.** There's no `firebase-tools` command to confirm Google/Email Link sign-in providers are enabled — this has to be visually confirmed in the console (Authentication → Sign-in method) before wiring auth code against it.
- When a prerequisite can't be verified programmatically, stop and report rather than guessing — this caught the missing-Firestore case before any rules were written against a nonexistent database.

## Firestore security rules
- **Rules can only `get()` a document by a known path — they cannot run queries.** If a rule needs to resolve "who is this caller" and the caller's auth UID isn't the document ID you need to check (e.g. `playerId` isn't the same as `authUid`, since the roster pre-exists sign-in), you need a reverse-index collection (`playerLinks/{authUid} -> playerId`) or custom claims via a Cloud Function. The reverse index is the lighter option when you don't already need a Cloud Function for claims.
- Centralize repeated identity checks (`isAdmin()`, `isPlayerRole()`) as rule helper functions rather than repeating the `playerLinks` lookup in every rule.
- Validate rules with `firebase deploy --only firestore:rules --dry-run` before a real deploy.

- **Firestore rules can't hide individual fields.** Data with a different audience needs its own collection (`socialPlans` vs `availability`).
- **Store per-week docs with explicit `weekId`/`playerId` fields** even when the composite doc ID encodes them. Doc-ID prefix lookups work but are a workaround.

## Planning and data
- **Simulate a proposed constraint against real data before building on it.** The "max−min Elo ≤ 75" rule would have left players unplaced in about half of weeks.
- **Verify workbook columns against their source before migration.** MAIN's "S3 ELO" held Season 2 final Elos, not the regressed start values.

## Pairing / cost-model design
- **Zero isn't a bonus.** A preference that only removes a penalty ties with unused options; give it a real negative cost.
- **Negative costs break branch-and-bound pruning** that assumes partial costs only grow. Shift costs per group to a minimum of 0 for the search and add the offset back.
- **Slot assignment is per group**, so one player's preference competes with three groupmates' rotation costs. Surface unmet preferences to the admin rather than over-weighting them.
- Keep the engine pure (no Firestore) so it can be unit-tested and re-run deterministically; put the Firestore loading/writing in a separate callable.

## Emulator and environment
- Use a `demo-` project ID for emulator work so nothing can reach real Firebase resources. Set the emulator env vars before `firebase-admin` loads, and reset emulator state at the start of each run.
- The Firestore emulator needs JDK 21+ (firebase-tools 15.x). After installing Java, set User PATH and `JAVA_HOME` with `[Environment]::SetEnvironmentVariable` (not `setx`) and fully restart VS Code — a new terminal tab inherits VS Code's old environment.
- `node --test pairing/` (directory argument) fails on Node 24; the bare `node --test` works on Node 20 and 24.

## Elo/scoring logic
- Not yet built. When porting the Elo formulas from the existing Excel workbook, watch for the same edge cases that came up in the original VBA macro build: merged cells, subs playing for regulars, duplicate player initials, and per-week (not per-set) Elo basis.
- Scores are last-write-wins and editable until an explicit "Lock Week" action — Elo must never be computed off `reported`-but-unlocked scores.

## Firebase / PWA
- See `.claude/skills/pwa-firebase-rules/SKILL.md` for the general Firestore/service-worker/Cloud Functions rules (Firestore refs through `firestorePaths.js`, `persistentLocalCache` not `enableIndexedDbPersistence()`, one Firebase project per app, etc.).
