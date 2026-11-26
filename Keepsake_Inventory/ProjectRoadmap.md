# Keepsake Inventory — Project Roadmap

**Last updated:** 2026-11-26 (Session 1)

## Phase 0 — Kickoff [IN PROGRESS]
- [x] Single-file prototype built and tested on iPhone (reference only, KI-002)
- [x] Claude Project "Keepsake Inventory" created; local folder `C:\DEV_Projects\Keepsake_Inventory` created
- [x] 7 standard docs drafted
- [ ] Confirm repo name (KI-003); create private repo `Keepsake_Inventory` under `Whit19`
- [ ] Commit the 7 docs to `dataforge-standards/Keepsake_Inventory/`
- [ ] Paste the Project Instructions snippet into the Claude Project
- [ ] Notion Projects row (Status Planning), local folder and repo URL added
- [ ] `.gitignore` covers `.env`, `node_modules`; confirm no secrets in any committed file
- [ ] Copy `pwa-firebase-rules` into `.claude/skills/`
- [ ] Rescue prototype data with Export data (.json) (KII-002)

## Phase 1 — Discovery / Requirements
- [x] Fields, decision flow, category model, set handling, photo and zoom requirement (KI-004 – KI-008)
- [ ] Auth model: individual logins or one shared password (KI-009)
- [ ] Roles: who can edit and delete
- [ ] "Keep in family" recipient list
- [ ] Whether appraisers / estate-sale helpers need a printable or CSV export

## Phase 2 — Core build [PLANNED]
- [ ] Separate Firebase project; Vite/React scaffold; Firebase SDK init; `firestorePaths.js` / `constants.js`; PWA manifest, service worker, icons
- [ ] Auth with closed sign-up; Firestore and Storage rules
- [ ] Item create / edit / delete with category and sub-category dropdowns, quantity, maker's mark, Antique checkbox
- [ ] Photo capture, client-side resize, upload to Storage; measured-pictures group
- [ ] Card photo carousel and full-screen viewer with pinch-zoom
- [ ] Decision flow: default Needs Review, per-decision follow-ups, decision date, quick change from the list
- [ ] Decision and category filters with counts
- [ ] Manage-categories panel (rename, delete, reorder, duplicate, bulk add) and family-member list
- [ ] Inline confirmations only (no `confirm()` / `prompt()`)
- [ ] Export (JSON)

## Phase 3 — Migration
- [ ] One-time import script: prototype JSON → Firestore items and taxonomy, photos → Storage files
- [ ] Spot-check item count and a sample of photos after import

## Phase 4 — Polish / launch [PLACEHOLDER]
- [ ] Install walkthrough for each family member (Safari → Add to Home Screen)
- [ ] Offline and weak-signal behavior checked inside the house
- [ ] Add family members as users and hand off
