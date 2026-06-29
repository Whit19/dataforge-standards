# ProjectRoadmap — knowledge-hub
Last updated: 2026-06-09

---

## Module 1 — Ask the Docs

### Phase 1 — Core Build [$1,200 – $1,800]

**Goal:** Working app deployed to Railway. Team can upload any PDF white paper or supplier doc and ask plain-English questions.

| Task | Status |
|------|--------|
| Express server + file upload (multer) | ✅ Done |
| PDF upload via Anthropic Files API | ✅ Done |
| DOCX support via mammoth | ✅ Done |
| Multi-question sessions (file_id reuse) | ✅ Done |
| MicroSynergies branded UI | ✅ Done |
| Logo fix (base64 data URI) | ✅ Done |
| Files API 404 bug fixed | ✅ Done |
| Token usage display | ✅ Done |
| Suggested question buttons | ✅ Done |
| Deploy to Railway (public URL) | ⬜ Todo |
| Test with Brad's FAQ document (PDF) | ⬜ Todo |
| Customize suggested questions for client | ⬜ Todo |
| Hand off + client walkthrough | ⬜ Todo |

### Phase 2 — Document Library [$900 – $1,400]

| Task | Status |
|------|--------|
| Admin panel — upload/remove/list documents | ⬜ Planned |
| Filter by supplier / organism / category | ⬜ Planned |
| Multi-document query (ask across several PDFs) | ⬜ Planned |
| Answer includes source document + page reference | ⬜ Planned |
| Redis session store (survive server restarts) | ⬜ Planned |

### Phase 3 — Integrations [$800 – $1,200]

| Task | Status |
|------|--------|
| Google Drive connector — auto-sync new PDFs | ⬜ Planned |
| Scheduled re-sync on Drive changes | ⬜ Planned |
| Export answers as formatted summaries | ⬜ Planned |

### Retainer [$150 – $250/month + API costs]

Updates, fixes, hosting support. API costs pass-through at actual cost.

---

## Module 2 — Tech Team Wiki

### Phase 1 — Core Build [$1,500 – $2,000]

**Goal:** Live, structured wiki accessible to the MicroSynergies team. Built from Brad's FAQ doc. Dale takes over content ownership at handoff.

| Task | Status |
|------|--------|
| Review FAQ doc — content structure decided | ✅ Done |
| Build demo with real FAQ content (v4) | ✅ Done |
| Fix quote scope language (organisms/SOPs → actual FAQ categories) | ⬜ Todo |
| Kickoff call with Brad (+ Dale if available) | ⬜ Todo |
| Check Dale's progress (if any) | ⬜ Todo |
| Decide hosting and stack | ⬜ Todo |
| Confirm auth requirement (basic auth vs individual logins vs none) | ⬜ Todo |
| Scaffold wiki structure from decided content categories | ⬜ Todo |
| Populate initial content from FAQ doc | ⬜ Todo |
| Configure navigation sidebar | ⬜ Todo |
| Build in-browser edit UI (Edit button → textarea → Save) | ⬜ Todo |
| Wire save route → write .md file + GitHub API commit | ⬜ Todo |
| Auth gate on edit routes (only authenticated users can save) | ⬜ Todo |
| Deploy to Railway | ⬜ Todo |
| Walkthrough edit workflow with Dale | ⬜ Todo |
| Team review and feedback round | ⬜ Todo |
| Handoff to Dale — content ownership | ⬜ Todo |

### Phase 2 — Enhancements [$800 – $1,200]

| Task | Status |
|------|--------|
| Full-text search (Typesense or Algolia) | ⬜ Planned |
| Tag and filter by category | ⬜ Planned |
| Rich-text editor upgrade (replace plain textarea) | ⬜ Planned |
| Version history UI — browse and restore prior page versions | ⬜ Planned |

### Phase 3 — AI Layer [$600 – $900]

| Task | Status |
|------|--------|
| Index wiki content for AI querying | ⬜ Planned |
| "Ask the Wiki" interface (reuses Ask the Docs server) | ⬜ Planned |
| Answers with page/section citations | ⬜ Planned |

### Retainer [$200 – $350/month]

Updates, fixes, small additions, hosting support, coordination with Dale on content questions.

---

## Future — RFQ Repository (Phase 2+)

> Do not quote until after a dedicated scoping session with Brad and Roy.

Known scope elements from Brad's notes:
- Raw material prices
- Pricing / quote templates
- Research / dosage / application templates
- Product charts / graphs / visualizations
- Strain data
- Export to PDF — internal dossier and external proposal format
- Data integrations: NetSuite, HubSpot, Excel, PDF white papers
