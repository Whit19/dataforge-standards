# IssuesTracker — knowledge-hub
Last updated: 2026-06-09

---

## Open

### KH-001 — Dale's wiki progress is unknown
**Status:** Open
**Description:** Brad mentioned Dale has the wiki as a 2026 incentive goal. It's unclear whether he's made any progress. Need to determine this at kickoff to avoid duplicating work or creating conflict.
**Resolution:** Ask Brad to connect with Dale before or during kickoff. Assess what (if anything) exists.

### KH-002 — Wiki stack and hosting not yet decided
**Status:** Open
**Description:** Docusaurus vs Next.js, Typesense vs Algolia, GitHub Pages vs Netlify vs Vercel — none decided yet.
**Resolution:** Decide at kickoff after reviewing FAQ doc scope and Dale's technical comfort level.

### KH-003 — Auth requirement not yet confirmed
**Status:** Open
**Description:** HTTP basic auth (shared password) vs individual logins vs no auth — pending kickoff discussion.
**Resolution:** Confirm with Brad at kickoff. Recommendation: HTTP basic auth for Phase 1 (~10 lines of code).

### KH-004 — Quote scope bullet needs correction
**Status:** Open
**Description:** Current quote has a bullet referencing "organisms, processes, SOPs" — doesn't reflect actual FAQ content categories (Sales, Products & Capabilities, Regulatory & Markets, Systems, Content & Tools).
**Resolution:** Update quote language before sending to Brad.

### KH-006 — CNAME record pending
**Status:** Open
**Description:** Custom domain `hub.microsynergies.com` added to Railway but DNS CNAME record not yet added by MicroSynergies IT. Site not live on custom domain yet.
**Resolution:** Send CNAME values from Railway to MicroSynergies IT to add to DNS.

### KH-007 — Brad's Anthropic API key not yet received
**Status:** Open
**Description:** Brad has instructions to create an Anthropic account, set a $50/month usage cap, and send the API key securely. Key not yet received.
**Resolution:** Follow up with Brad. Once received, add to Railway env vars and replace the DataForge key.

---

## Deferred

### KH-005 — In-memory sessions reset on server restart
**Status:** Deferred
**Description:** Session store lives in process memory. Any server restart (deploy, crash, Railway sleep) wipes all active sessions. Users must re-upload their document.
**Resolution:** Acceptable for Phase 1. Redis session store planned for Phase 2.

---

## Resolved

### KH-006 — Files API returns 404 not_found_error on upload
**Status:** Resolved
**Description:** Every call to `anthropic.beta.files.upload()` returned a 404 `not_found_error` despite valid API key and correct endpoint.
**Resolution:** The `betas` parameter was being passed as an HTTP header override instead of inside the call params object. See MS-004 in DecisionLog and BestMethods.md.

### KH-007 — MicroSynergies logo renders as broken/partial characters
**Status:** Resolved
**Description:** Inlining the MicroSynergies SVG paths directly in HTML caused broken rendering — partial letters and missing colors.
**Resolution:** SVG file is ~16KB. Streaming renderer truncates inline SVG at that size. Fixed by embedding as a base64 data URI in an `<img>` tag. See MS-005 in DecisionLog.

### KH-008 — DOCX Word document validation errors on generated .docx files
**Status:** Resolved
**Description:** The `docx` Node.js library generates `w:tcBorders` XML with incorrect element order and old element names where newer OOXML schema expects `start`/`end`. Caused validation failures.
**Resolution:** Post-process the generated ZIP with Python. See BestMethods.md for full fix.

### KH-009 — .env setup confusion (cp step)
**Status:** Resolved
**Description:** Initial README instructed users to `cp .env.example .env`, creating confusion about which file to edit.
**Resolution:** Removed the `cp` step. Instruct users to create `.env` directly with a text editor.
