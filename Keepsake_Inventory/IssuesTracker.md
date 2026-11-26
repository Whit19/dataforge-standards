# Keepsake Inventory — Issues Tracker

**Last updated:** 2026-11-26 (Session 1)

## Open

### KII-002 — Prototype data may exist only inside the Claude artifact's storage
**Status:** Open (logged 2026-11-26)
**Description:** Items entered while testing the prototype live in Claude artifact storage, which behaved inconsistently: in some views the list came up empty with "Could not load saved items," and saves failed in others. Nothing has been confirmed lost, but nothing is backed up.
**Resolution:** None yet. Run Export data (.json) in any view that still lists the items, and keep the file as the migration input (KI-010).

### KII-003 — The prototype uses `confirm()` and `prompt()`, which a standalone iOS PWA blocks
**Status:** Open (logged 2026-11-26)
**Description:** Delete-item, remove-category and duplicate-category use browser dialogs. They work in a browser tab but are blocked in an installed PWA (Master Protocol Section 9).
**Resolution:** None yet. The rebuild must use inline confirmation UI everywhere.

## Deferred

### KII-001 — Prototype storage unreliable: saved items not loading, "Save failed" on add and edit
**Status:** Deferred (logged 2026-11-26)
**Description:** Saving worked in some views of the published artifact and failed in others; the same artifact sometimes showed no saved items. Cause not confirmed from inside the artifact.
**Resolution:** Not being fixed — superseded by the Firebase rebuild (KI-002).

### KII-004 — A web app cannot launch Apple's Measure app or read its result back
**Status:** Deferred (logged 2026-11-26) — platform limitation
**Description:** There is no hook between a web page and the Measure app.
**Resolution:** Workaround in use (KI-007): take the measured photo in Measure, then add it from the photo library under Measured pictures.

## Resolved
(none yet)
