# TNPL — Time Log

| Date | Session summary |
|---|---|
| 2026-09-22 | Kickoff: stack, repo, Notion project row, 7 standard docs scaffolded, Project Instructions snippet drafted. |
| 2026-09-22 | GitHub repo created, .gitignore added, pwa-firebase-rules skill copied in, secrets check clean. |
| 2026-09-22 | Discovery: workbook + process notes reviewed, data model drafted, time-slot/court structure confirmed, pairing algorithm approach designed, v1 scope (full loop) decided. CC scaffolded the Vite/React app + Firebase SDK init + functions/ skeleton. |
| 2026-09-22 | Auth/permissions design: sign-in method, score-submission model, season-enrollment vs roster distinction, staff role, score-lock flow. Firebase project `tnpl-pwa` created; Firestore + Google/Email Link providers enabled. CC implemented auth.js, firestore.rules (incl. playerLinks reverse index), and page shells with route guards. |
| 2026-09-23 | Pairing engine designed, built and verified end-to-end against the Firebase emulators (pure engine + admin-only `generatePairings` callable, per-set partner rotation, `pairing_draft` status, `socialPlans`, new rules); found and fixed the MAIN S3 Elo column and a preferred-slot spec gap. |
| 2026-09-25 | Process review and full mockup set in the "TNPL Mockups" design canvas: palette C (court blue), logo A, compact pairing-draft layout A. Mockup-first-then-code rule adopted for every page. |
| 2026-09-28 | Large session: pairing engine finished (publish/hold/finalize, weekly automation), season setup, admin dashboard and pairing-draft/edit screens; batch 4 (roster admin, invite email v1, decline page, join requests, profile, season sign-up); batch 5 (invite email v2 "install-first," iPhone install gate); batch 6 (bottom tab bar, Home season card, Coming Soon pages). First deploy of rules/functions/hosting. Investigated and fixed a roster status bug (`sendInvites` downgrading a signed-in player) with a one-time backfill; corrected 12 players' phone numbers via a one-time script. Test suite grew to 115 (105 functions + 10 client). |
