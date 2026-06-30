# DecisionLog — knowledge-hub
Last updated: 2026-06-30

> Append-only. Never delete or edit past entries. Mark superseded decisions as `Superseded by [ID]`.

---

### MS-001 — PDF preferred over DOCX for uploads
**Decision:** Recommend clients convert Word docs to PDF before uploading. Accept DOCX as fallback via mammoth text extraction.
**Rationale:** Anthropic API has native PDF support as a first-class document type. PDF includes per-page image parsing, citation support, and higher accuracy on formatted documents. DOCX extraction loses formatting and images.
**Alternatives considered:** DOCX-only (loses images), PDF-only (too restrictive for clients).
**Status:** Active

---

### MS-002 — Standard Anthropic API, not Zero Data Retention
**Decision:** Use standard Anthropic API for Phase 1. Do not pursue ZDR contract.
**Rationale:** Standard API deletes inputs/outputs within 7 days and never uses them for training. Sufficient for Phase 1 white paper content. ZDR requires Anthropic enterprise contract — appropriate to revisit when RFQ Repository is scoped, as it may handle more sensitive pricing data.
**Alternatives considered:** ZDR enterprise contract (overkill for Phase 1), local/self-hosted model (too much infrastructure).
**Status:** Active

---

### MS-003 — In-memory session store for Phase 1
**Decision:** Store upload sessions in a plain JavaScript object in process memory. No Redis or database in Phase 1.
**Rationale:** Simple, zero dependencies, sufficient for single-user demo and initial client testing. Sessions are inherently short-lived.
**Alternatives considered:** Redis (adds infrastructure complexity), SQLite (unnecessary for transient data).
**Status:** Active — Redis planned for Phase 2

---

### MS-004 — betas param inside SDK call, not as HTTP header
**Decision:** Pass `betas: ["files-api-2025-04-14"]` as a param inside the SDK method call object, not as a second argument headers override.
**Rationale:** Passing as header override causes a 404 `not_found_error`. The SDK expects betas inside the call params. Discovered through debugging a persistent 404 error.
**Alternatives considered:** Header override (broken), omitting betas (Files API not activated).
**Status:** Active — see BestMethods.md for canonical pattern

---

### MS-005 — MicroSynergies logo as base64 data URI
**Decision:** Embed the MicroSynergies SVG logo as a base64-encoded data URI in an `<img>` tag rather than inlining SVG paths directly in HTML.
**Rationale:** The SVG file is ~16KB of path data. When inlined directly, the streaming renderer truncates the content and the logo renders broken. Base64 data URI in an `<img>` tag renders correctly.
**Alternatives considered:** Inline SVG (broken at this file size), external URL (not self-contained).
**Status:** Active

---

### MS-006 — Single repo (knowledge-hub) instead of two separate repos
**Decision:** Consolidate ask-the-docs and tech-team-wiki into one repo named knowledge-hub. Renamed from ask-the-docs (which held all working code).
**Rationale:** Both modules share one Express server, one Railway deployment, one .env, and the same branding. Phase 3 of the wiki explicitly reuses the Ask the Docs architecture. Separate repos created split-brain MD files and artificial separation for what is one product. Quoting remains per-module — repo structure doesn't affect commercial scope.
**Alternatives considered:** Two separate repos (creates maintenance overhead, couples two tools that share infrastructure).
**Status:** Active

---

### MS-W002 — Collaborate with Dale, don't replace his effort
**Decision:** Tom builds the platform and populates initial content from Brad's FAQ doc. Dale owns ongoing content. If Dale has made progress, evaluate building on it rather than starting fresh.
**Rationale:** Dale has the wiki as a 2026 incentive goal. Replacing his work entirely undermines his incentive structure. This is a collaborative arrangement that keeps Dale engaged and reduces Tom's ongoing content burden.
**Alternatives considered:** Tom owns all content (too much ongoing work for retainer price), Dale builds it alone (unlikely to happen).
**Status:** Active

---

### MS-W003 — Wiki stack decision deferred to kickoff call
**Decision:** Final stack details (search, hosting) will be confirmed at the kickoff call. Core architecture is now decided: custom Express server with in-browser editing (see MS-W004).
**Rationale:** Can't finalize search and hosting without knowing content scale and Dale's technical comfort level.
**Alternatives considered:** Decide now (premature without content audit).
**Status:** Active — update when confirmed at kickoff

---

### MS-W004 — Wiki uses custom in-browser editing, not a static site generator
**Decision:** Build a lightweight edit UI directly into the Knowledge Hub Express server. Dale edits wiki pages in-browser via a textarea. Server writes the .md file and commits to GitHub via the GitHub API. No Docusaurus, no external CMS.
**Rationale:** The demo already showed in-browser editing — delivering less than the demo creates a poor client experience. Dale should not need a GitHub account or any external tooling to maintain content. Since the wiki already runs on Express (same server as Ask the Docs), building the edit routes is straightforward and keeps everything in one deployable unit.
**Alternatives considered:** Docusaurus + GitHub web editor (requires Dale to use GitHub), Decap CMS (extra setup, external dependency), Notion sync (adds pipeline complexity).
**Status:** Active

---

### MS-007 — MicroSynergies declines proposal, builds in-house
**Decision:** MicroSynergies will not move forward with the Knowledge Hub proposal (Ask the Docs + Tech Team Wiki). They will build both capabilities internally with their own team.
**Rationale:** Per Brad (CTO, email 2026-06-30): after reviewing the demos (Wiki + White Paper AI Chat), the team realized they were farther along internally than initially thought and want to give their own team the opportunity to build these capabilities over the coming months. This was a build-vs-buy decision, not a budget objection — Brad noted the demo was valuable specifically because it "helped clarify what 'good' looks like." Brad explicitly left the relationship open: he will reach back out if internal progress stalls or if another project arises that's a better fit for outside collaboration, and offered to make referral introductions to other companies that could benefit from DataForge's work.
**Alternatives considered:** Phased delivery starting with Ask the Docs only, retainer renegotiation, full approval at original quote — none of these were the actual blocker; client chose in-house build over all three.
**Status:** Active — project status set to Inactive in SessionStarter.md. Revisit if MicroSynergies re-engages or a referral materializes.
