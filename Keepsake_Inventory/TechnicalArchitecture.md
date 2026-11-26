# Keepsake Inventory — Technical Architecture

**Last updated:** 2026-11-26 (Session 1 — draft; nothing below is implemented yet)

## Stack
- Frontend: React + Vite, PWA (installable Home Screen app on iPhone; Safari → Share → Add to Home Screen)
- Backend / data: Firebase — Firestore (records), Firebase Storage (photo files), Firebase Auth (login), Firebase Hosting (deploy)
- Firebase project: [TBD — separate project for this app, KI-012]
- Repo: https://github.com/Whit19/Keepsake_Inventory (private) — [pending creation, KI-003]
- Local folder: `C:\DEV_Projects\Keepsake_Inventory`
- Docs: `dataforge-standards/Keepsake_Inventory/`
- Prototype reference file: `reference/keepsake-inventory.html` inside the code repo (copy of the Claude artifact; read-only reference)

## Scope for v1
Photograph, describe, categorize and decide on each item, from a phone, with a simple login; share the list with family. Everything in "Carry over from the prototype" below.

## Data model (Firestore) — draft
All refs go through `firestorePaths.js`; all constants (collection names, decision ids, labels) live in `constants.js` (Master Protocol Section 8).

**`items/{itemId}`**

| Field | Type | Notes |
|---|---|---|
| `name` | string | Shown as "Description" in the UI |
| `quantity` | number | integer ≥ 1, default 1; a set is one record (KI-004) |
| `category`, `subcategory` | string | Pick from the taxonomy (KI-006) |
| `makerMark` | string | Maker's mark / brand, free text (KI-007) |
| `isAntique` | boolean | Per-item checkbox, available in every category |
| `decision` | string | `review` (default) · `donate` · `sell` · `keep` · `use` |
| `decisionDate` | timestamp | Set whenever `decision` changes; shown on the card |
| `decisionDetail` | map | `donate`: `where` · `sell`: `amount` (free text), `needsAppraisal` (bool) · `keep`: `memberId` or `otherName` · `use`: `timing` (`current` or `after`) |
| `notes` | string | Condition, story, appraisal notes |
| `photos` | array of strings | Storage paths, ordered |
| `measurePhotos` | array of strings | Storage paths for Measure-app screenshots |
| `createdAt`, `createdBy`, `updatedAt`, `updatedBy` | timestamp / uid | Audit fields |

**`settings/taxonomy`** — one ordered list: `[{ name, subcategories: [string] }]`. Order is the display order (reorderable).
**`settings/familyMembers`** — ordered list of people an item can be kept for. Configurable data, not hardcoded labels (KI-011).
**`users/{uid}`** — role and display name [shape TBD with KI-009].

## Photos (KI-008)
- Stored as image files in Firebase Storage at `items/{itemId}/{photoId}.jpg`; Firestore holds only the path. Firestore documents cap at about 1 MiB, so base64 photos do not scale.
- Resize on the device before upload: long edge about 1600 px, JPEG quality about 0.82 (the prototype's setting) — large enough to read a hallmark when zoomed.
- Full-screen photo viewer: swipe between photos (regular and measured), pinch/double-tap to zoom in and out.
- On the item list, each card's photo area swipes through all its photos.
- Uploads need connectivity — queue and retry rather than lose a photo on weak signal.

## Auth & roles (KI-009 — open)
- Firebase Auth, sign-up closed: an admin adds each family member.
- Firestore and Storage rules: signed-in family members only; no public read.
- Individual logins vs one shared password is undecided; individual logins enable "who changed what."

## PWA / iPhone requirements
- Follow Master Protocol Section 9: `injectManifest` + `rollupFormat: 'iife'`, no-cache for `index.html` and service-worker files, immutable hashed assets, white-background icons at 180, 192 and 512 px, `maximum-scale=5.0`.
- No `window.confirm()` / `prompt()` — blocked in a standalone iOS PWA. Use inline confirmation UI (KII-003).
- Photo capture uses a file input with `capture="environment"`; the measured-photo input omits `capture` so it opens the photo library.
- Firestore uses `persistentLocalCache`; `setLoading(false)` must not sit behind an awaited profile load (Section 8).

## Export / migration (KI-010)
- The prototype's **Export data (.json)** writes `{ exportedAt, itemCount, items, taxonomy }`; photos are base64 data URLs inside each item.
- A one-time import script decodes each photo to a Storage file, writes items to Firestore, and loads the category list.
- The production app keeps its own export (JSON, with a printable or CSV view still an open question).

## Carry over from the prototype
- Fields: category / sub-category dropdowns, antique checkbox, photos, measured pictures, description, quantity, maker's mark, decision, notes.
- Manage-categories panel: rename (cascades to items), delete, reorder with up/down arrows, duplicate a category with its sub-categories, comma-separated bulk add, with a visible Add button (the Enter key is unreliable on mobile).
- List: two dropdown filters (decision with counts, category); cards with swipeable photos, an "×N" quantity badge, an Antique tag, an Edit button, and a quick decision dropdown with a "since [date]" stamp.
- Checkbox-style toggles for Antique and Needs appraisal.
- On save failure, show the actual error reason and the record size, not a generic message.
