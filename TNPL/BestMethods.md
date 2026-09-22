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

## Elo/scoring logic
- Not yet built. When porting the Elo formulas from the existing Excel workbook, watch for the same edge cases that came up in the original VBA macro build: merged cells, subs playing for regulars, duplicate player initials, and per-week (not per-set) Elo basis.
- Scores are last-write-wins and editable until an explicit "Lock Week" action — Elo must never be computed off `reported`-but-unlocked scores.

## Firebase / PWA
- See `.claude/skills/pwa-firebase-rules/SKILL.md` for the general Firestore/service-worker/Cloud Functions rules (Firestore refs through `firestorePaths.js`, `persistentLocalCache` not `enableIndexedDbPersistence()`, one Firebase project per app, etc.).
