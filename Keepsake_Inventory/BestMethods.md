# Keepsake Inventory — Best Methods

**Last updated:** 2026-11-26 (Session 1)

Hard-won lessons for this project. Read before writing any code.

## Prototype lessons (carry into the rebuild)
- **A Claude artifact's built-in storage is not a production datastore.** Saving needs a Claude account for every viewer, there is no real login, and storage behaved inconsistently between views. Keep real data in Firebase; use an artifact only to prototype a layout.
- **Always give the person an export before trusting a store.** The Export data (.json) button was added so entered data could survive a platform change.
- **Show the real error.** A generic "Save failed — try again" hid the cause. Surface the actual error message and the record size.

## Mobile UI
- **Do not rely on the Enter key to submit on a phone.** Give every add-field a visible Add button; keep Enter as a bonus.
- **Accept comma-separated input** where a person adds several things at once (sub-categories, categories); skip duplicates and say how many were skipped.
- **Do not make the whole card tappable if it contains a swipeable gallery.** Tap and swipe conflict; use an explicit Edit button.
- **Toggles that mean yes/no should look like checkboxes** (box plus checkmark). Reserve colored dots for values that carry meaning elsewhere, such as the decision status.

## Photos
- **Do not store photos as base64 inside Firestore documents.** Documents cap at about 1 MiB. Store files in Firebase Storage and keep the path in the record.
- **Resize on the device before upload,** long edge about 1600 px, JPEG quality about 0.82 — small enough to upload on weak signal, large enough to read a hallmark when zoomed.
- **Two inputs, two behaviors:** the camera input uses `capture="environment"`; the measured-picture input omits it so it opens the photo library.

## iPhone PWA
- `window.confirm()` and `prompt()` are blocked in a standalone iOS PWA. Use inline UI (Master Protocol Section 9).

## Process
- **dataforge-standards is public.** No personal names, locations, values or dollar figures in any doc (Sections 5c and 6a). Anything family-specific is data in Firestore, not text in a doc.
- **Do not edit the prototype further.** New behavior goes in the rebuild; fixes to the artifact only matter if they protect data (export).
