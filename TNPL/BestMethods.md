# TNPL — Best Methods

**Last updated:** 2026-09-30

Hard-won lessons for this project. Read before writing any code.

## Firebase provisioning
- **A fresh Firebase project does not auto-provision Firestore.** `firebase-tools` has no command to create the database, only to interact with one that already exists (`firebase firestore:databases:list` returns "No databases found" until you create it manually via console → Firestore Database → Create database). Always verify with that command before writing/deploying rules, rather than assuming the console setup is done.
- **Auth provider status isn't checkable via CLI/API.** There's no `firebase-tools` command to confirm Google/Email Link sign-in providers are enabled — this has to be visually confirmed in the console (Authentication → Sign-in method) before wiring auth code against it.
- When a prerequisite can't be verified programmatically, stop and report rather than guessing — this caught the missing-Firestore case before any rules were written against a nonexistent database.
- **Every new callable/HTTP function needs "Allow public access" set manually in the Cloud Run console after deploy.** Use the exact exported function names from `firebase functions:list` (or the deploy output) — a prompt's guessed name (e.g. `declineInvite`) can be wrong; the real deployed name (`decline`) is what actually needs the Cloud Run change.
- **New Firestore composite indexes need their own deploy and a wait.** `firebase deploy --only firestore:indexes` reports success as soon as the definition is accepted, not once the index finishes building. Poll the Firestore Admin REST API (or the console) for `READY`/"Enabled" before testing a feature that depends on the index — testing against a still-`CREATING` index looks like a query bug that isn't one.
- **Service-account keys: one per task.** Create a fresh key, store it outside the repo, hand Claude only the file path (never paste key contents into chat), then delete the file **and** revoke the key in Google Cloud once the task is done. Don't reuse a key across unrelated one-off scripts.

## Firestore security rules
- **Rules can only `get()` a document by a known path — they cannot run queries.** If a rule needs to resolve "who is this caller" and the caller's auth UID isn't the document ID you need to check (e.g. `playerId` isn't the same as `authUid`, since the roster pre-exists sign-in), you need a reverse-index collection (`playerLinks/{authUid} -> playerId`) or custom claims via a Cloud Function. The reverse index is the lighter option when you don't already need a Cloud Function for claims.
- Centralize repeated identity checks (`isAdmin()`, `isPlayerRole()`) as rule helper functions rather than repeating the `playerLinks` lookup in every rule.
- Validate rules with `firebase deploy --only firestore:rules --dry-run` before a real deploy.

- **Firestore rules can't hide individual fields.** Data with a different audience needs its own collection (`socialPlans` vs `availability`). A "hide my contact info" toggle is cosmetic unless the contact data actually moves to its own server-written collection and the original collection's reads are locked down — otherwise anyone who can read the original doc still sees the hidden field (TP-056).
- **Locking a collection's client reads down must be the last deploy, after every screen that used to read it is live on the new build.** If the rules file holds both a new collection's rules and an old collection's lock-down in the same change, split them into separate commits/deploys — deploying the lock before the new hosting build ships would break every still-live screen reading the old collection directly (TP-057).
- **Store per-week docs with explicit `weekId`/`playerId` fields** even when the composite doc ID encodes them. Doc-ID prefix lookups work but are a workaround.

## Firestore client SDK
- **`serverTimestamp()` isn't allowed inside array elements.** A per-set `savedAt` inside `matchGroups.sets[]` has to use `Timestamp.now()` instead.
- **Once an `onSnapshot` error callback fires, that listener is dead for good — no further callbacks ever arrive.** Never treat the error callback the same as a completed read that found no document; show a distinct error (ideally with the error code), or a permission hiccup or transient network blip permanently looks like "this doesn't exist" (ISS-016).

## Planning and data
- **Simulate a proposed constraint against real data before building on it.** The "max−min Elo ≤ 75" rule would have left players unplaced in about half of weeks.
- **Verify workbook columns against their source before migration.** MAIN's "S3 ELO" held Season 2 final Elos, not the regressed start values.
- **A source workbook's display rounding can exceed a validation tolerance that's tighter than the precision the workbook actually stores.** The Season 25-26 import workbook displayed Elo before/after at 1 decimal but set-level changes at 2; validate against the precision the source actually has, not an arbitrarily tight number, and widen the tolerance rather than treating a workbook-rounding mismatch as a script bug (ISS-022).
- **When adding a second season's data, audit every query that touches seasons/weeks/matchGroups/eloHistory for season scoping first.** An "any locked match ever" check (the starting-Elo lock guard) broke silently once a second season's locked matches existed — it would have blocked every returning player from a legitimate correction. Do the scoping audit before the import, not after something breaks (ISS-023).
- **Derive a status field from a fact, not from the last action taken.** `inviteStatus` alone drifted out of sync with reality because a later action (re-sending an invite) could overwrite it; `firstSignInAt`, set once and never re-set, is the more durable source of truth for "has this player signed in." When a value can legitimately only move forward, prefer deriving state from it over a mutable status string that any code path can overwrite.
- **A field that's read but never written anywhere hides a bug instead of surfacing one.** `season.menuUrl`/`golfSimUrl`/`weeklySpecial` sat unused for a while before their real replacements arrived; remove dead fields as soon as a real source replaces them rather than leaving them readable (ISS-024).

## One-time data scripts
- **Dry run by default, `--apply` to write.** Every one-time correction/backfill script (phone-number fixes, sign-in backfill) defaults to a dry run that only prints what it would do; a separate flag is required to actually write.
- **Abort the whole run on any mismatch, don't skip-and-continue.** A script correcting specific records (matched by id + email) stops entirely and writes nothing if even one record doesn't match what's expected, rather than silently applying the rows that do match and skipping the rest.
- **Log names only, never contact info.** Phone numbers and emails are never printed to the console or written into a commit message, even for a script whose whole job is to fix them.
- **Private input files are gitignored and deleted after use**, alongside the service-account key used to run the script against the real project.
- **Running a one-time script against production from PowerShell:** set `$env:GOOGLE_APPLICATION_CREDENTIALS` in the *same* terminal session as the `node` command, and pass `--project tnpl-pwa` explicitly — then delete the key file and revoke the key in Google Cloud once the script has run.
- **A seed/demo script's own self-check writes can wipe hand-curated demo data.** If a script writes curated states (e.g. one match fully scored, another partially scored) and then runs a self-check that also writes to a match group, point the self-check at a spare, otherwise-untouched record — not one of the curated ones — and restore its shape afterward if the check has to write something.

## Email
- **HTML email needs table-based layout and inline styles, not a stylesheet or SVG.** Gmail/Outlook strip `<style>` blocks and block SVG; images must be plain PNGs referenced by absolute hosting URL (`https://.../email/...`), not relative paths or data URIs.
- **Build an email's "what the icon looks like" picture from the real app icon file, not a redrawn approximation.** A hand-drawn stand-in silently drifts from the actual icon the first time the real one changes.
- **A one-tap link in an email must never record its answer on page load.** Email security scanners open every link in an inbound email; a link that answers on load (e.g. "Yes, I'm in") would get silently recorded by the scanner, not the player. Land on a page that requires an explicit confirm tap instead.

## iPhone PWA
- **Safari and the installed Home Screen app keep completely separate storage and sign-in state.** Signing in while browsing in Safari does not carry over to the app once it's added to the Home Screen — a visitor has to sign in again from the icon. Any first-run flow aimed at iPhone needs to account for this explicitly (see the install gate) rather than assuming a session persists across that boundary.
- **iOS 26 Safari's default "Compact" layout hides the Share button behind •••.** Only the "Bottom" layout and iOS 18 or earlier show Share directly. Any "add to Home Screen" instructions need to lead with tapping ••• first, with a note for readers on an older layout/iOS version that Share may already be visible.
- **iOS (Safari and installed PWAs) blocks `window.open` called after an `await`.** The user-gesture context needed to open a new tab/window is lost by the time an awaited call (e.g. `getDownloadURL`) resolves. Fetch the URL ahead of time and render a real `<a target="_blank">` instead of calling `window.open` from inside an async handler (ISS-021).

## Pairing / cost-model design
- **Zero isn't a bonus.** A preference that only removes a penalty ties with unused options; give it a real negative cost.
- **Negative costs break branch-and-bound pruning** that assumes partial costs only grow. Shift costs per group to a minimum of 0 for the search and add the offset back.
- **Slot assignment is per group**, so one player's preference competes with three groupmates' rotation costs. Surface unmet preferences to the admin rather than over-weighting them.
- Keep the engine pure (no Firestore) so it can be unit-tested and re-run deterministically; put the Firestore loading/writing in a separate callable.

## Emulator and environment
- Use a `demo-` project ID for emulator work so nothing can reach real Firebase resources. Set the emulator env vars before `firebase-admin` loads, and reset emulator state at the start of each run.
- The Firestore emulator needs JDK 21+ (firebase-tools 15.x). After installing Java, set User PATH and `JAVA_HOME` with `[Environment]::SetEnvironmentVariable` (not `setx`) and fully restart VS Code — a new terminal tab inherits VS Code's old environment.
- `node --test pairing/` (directory argument) fails on Node 24; the bare `node --test` works on Node 20 and 24.
- **Check a production build isn't still pointing at the emulators after local testing.** `VITE_USE_EMULATORS=true` in `.env.local` is added for a walkthrough and must be removed before `npm run build`/deploy, or the deployed app tries to reach the local emulator ports.
- **Reseeding the emulator wipes Auth accounts, not just Firestore.** Sign out and back in after any reseed script runs, or reads silently fail against a now-orphaned session.
- **Testing an automated email a week early:** tapping an admin "send now" action (e.g. "Open availability now") marks that week's send as done, so the real Monday automation then skips it. Use a "send a test to me"-style action instead, which doesn't touch the week's state, or reset the week afterward with a dedicated reset script.

## Elo/scoring logic
- **Built (2026-09-29).** Pure engine `functions/elo/elo.js`, ported from the Excel workbook and verified against its worked example (two 1600/1500 players beating two 1450/1400 players 6-3 → about +9.7/-9.7 Elo, within 0.1). Margin multiplier: 0.6 + 0.16 × (|point diff| − 1). A player's Elo is fixed for the whole week (start-of-week basis) — every set they play, across both matches for a two-match player, uses the same start value, and the deltas are summed and applied once at lock (TP-046).
- Scores are last-write-wins and editable until an explicit "Lock Week" action — Elo must never be computed off `reported`-but-unlocked scores.
- **A re-read guard before a batch write does not stop a double-apply under real concurrency.** A manual "Lock Week" tap and a scheduled auto-lock can race; both can pass a plain re-read check before either commits. Run the whole read-compute-write sequence inside a Firestore **transaction** instead — its reads double as the write-conflict check, so a losing concurrent attempt is retried from scratch and sees the now-locked state, rather than blindly trusting a value that was fresh when read but stale by the time it wrote (TP-048).

## Process
- **Check the existing mockup canvas before proposing a new mockup.** Several pages built this session (Rules & league info, Home in-season cards, Settings) had already been mocked up and approved in an earlier session; re-proposing a fresh layout wastes a round trip Tom already spent.
- **A page that returns early for an unpublished/not-ready state hides everything below that early return, not just the obviously-gated part.** Matches' dinner/golf-sim section disappeared before pairings published because it sat below an early return meant only to gate the match list; check every early-return branch when a section "doesn't show" instead of assuming the bug is in the section's own code (ISS-020).
- **Before writing a prompt about "the X section," confirm which component actually renders it.** A prompt misidentified `SocialPlanEditor` (opens only from "Change my plans") as the home of the dinner/golf-sim links, which actually belonged in the always-visible section row — the links worked, just in the wrong place (ISS-019).
- **Reorganize a growing mockup/design canvas into chapters once tall boards start overlapping and changes get hard to find**, rather than letting it sprawl — the "TNPL Mockups" canvas was split into 11 chapter tabs this session for exactly that reason.

## Firebase / PWA
- See `.claude/skills/pwa-firebase-rules/SKILL.md` for the general Firestore/service-worker/Cloud Functions rules (Firestore refs through `firestorePaths.js`, `persistentLocalCache` not `enableIndexedDbPersistence()`, one Firebase project per app, etc.).
