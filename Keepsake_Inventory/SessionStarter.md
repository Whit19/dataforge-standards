# Keepsake Inventory — Session Starter
**Protocol:** Load MASTER_CLAUDE_PROTOCOL.md before this file.
Repo: github.com/Whit19/dataforge-standards

**Project:** Keepsake Inventory — a PWA for documenting and tracking family members' assets
**Status:** Kickoff / Planning. A single-file prototype exists (reference only). The production build has not started; the code repo, Firebase project and Notion row are not created yet.
**Last updated:** 2026-11-26 (Session 1 — kickoff)

## One-line description
A PWA that helps a family document household assets (starting with silver and furniture) — photos, measured photos, maker's marks — and decide what to donate, sell, keep in the family, or keep using, ahead of an older relative downsizing.

## Current status
- **Prototype (pre-kickoff):** a single-file HTML Claude artifact on Claude's shared `window.storage`. It proved the fields and flow (see TechnicalArchitecture, "Carry over from the prototype"). It is **reference only** — it needs a Claude account to save, has no real login, and its storage was unreliable (KI-002, KII-001).
- **Not started:** Firebase project, repo, auth, data model, migration of prototype data.
- Prototype data may exist only inside the artifact's storage — see KII-002.

## Key dates
- Downsizing is "soon"; no firm date recorded. Add one here when known — it sets how much of the inventory must be done first.

## Next priorities
1. Confirm repo name (KI-003); create the private repo and local folder; commit these 7 docs to `dataforge-standards/Keepsake_Inventory/`; paste the Project Instructions snippet into the Claude Project.
2. Create the Notion Projects row (Section 14f).
3. Rescue prototype data: run **Export data (.json)** in any view that still lists items (KII-002).
4. Create a separate Firebase project (KI-012); scaffold the Vite/React PWA; copy `pwa-firebase-rules` into `.claude/skills/`.
5. Settle the open questions below, then build Phase 2 in ProjectRoadmap.

## Key decisions so far
See DecisionLog.md — KI-001 through KI-012. Headlines:
- **KI-001:** React + Vite + Firebase, delivered as an installable iPhone PWA.
- **KI-002:** the Claude-artifact prototype is a reference, not the production path.
- **KI-005:** Needs Review default, then Donate / Sell / Keep in family / Still in use, each with its own follow-up and a decision date.
- **KI-006:** categories, sub-categories and the Antique flag are editable data.

## Open questions
- **Auth model (KI-009):** individual logins per family member (shows who changed what) or one shared password (simpler for the oldest user)?
- **Roles:** can every family member edit and delete any item, or only add and comment?
- **"Keep in family" recipients:** a configurable list of family members (assumed) — who is on it, and can an item go to more than one person?
- **Output for others:** do appraisers or estate-sale helpers need a printable/CSV export, in addition to the JSON export used for migration?
