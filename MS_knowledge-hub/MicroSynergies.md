# MicroSynergies — Company Reference
**DataForge LLC client reference document**
Last updated: 2026-06-08

---

## Company Overview

| Field | Detail |
|-------|--------|
| Company | MicroSynergies |
| Location | Glendale, WI |
| Industry | Microbial biotechnology |
| Size | 11–50 employees |
| Growth target | 28% annual revenue CAGR |
| Website | microsynergies.com |

MicroSynergies sources, formulates, and manufactures beneficial organisms across global markets. They hold a library of over 40,000 strains with 4,000 in production worldwide. Their extensive global supplier network drives much of their data complexity.

---

## Key Contacts

| Name | Role | Notes |
|------|------|-------|
| Brad | CTO | Primary champion. Technical. Decision-maker on all projects. |
| TC | Leadership | Co-OOO with Brad for conferences. Involved in budget decisions. |
| Dale | Tech team | Wiki is assigned as one of his 2026 incentive goals. Likely hasn't started. Collaborate — don't replace. |
| Roy | Operations / Sourcing | Involved in RFQ process. Sign-off required on orders >$10k alongside Brad. |

---

## Brand

| Element | Value |
|---------|-------|
| Primary blue | `#0057B7` |
| Light blue | `#00A1E1` |
| Red accent | `#BD3742` |
| Logo file | `microsynergies-logo.svg` — always embed as base64 data URI in `<img>` tag; never inline SVG paths (they get truncated) |

---

## Tech Stack (existing)

| Tool | Purpose |
|------|---------|
| HubSpot | CRM — has REST API |
| NetSuite | ERP — has REST API |
| Google Workspace | Docs, Drive, Email |
| Excel | Spreadsheets (supplier pricing, product data) |
| PDF white papers | Technical research library |
| T3 (internal name) | Tech team tooling — described as "used but not effective" |

---

## Active Projects (DataForge engagements)

| # | Project | Repo | Status |
|---|---------|------|--------|
| 1 | White Paper AI Chat ("Ask the Docs") | `ask-the-docs` | Phase 1 built, testing |
| 2 | Tech Team Wiki | `tech-team-wiki` | Phase 1 quoting |
| 3 | RFQ Repository | TBD | Scoping pending — needs dedicated discovery session |
| 4 | Simplified Playbook / RFQ Checklist | TBD | Phase 2, tied to RFQ Repository |

> See each project's `/docs` folder for detailed technical and session context.

---

## Pain Points (from Brad)

1. **Data silos** — supplier pricing, tech specs, and team tools don't communicate
2. **Institutional knowledge risk** — critical info lives in people's heads
3. **Inconsistent processes** — RFQ process varies by person; no standard checklist
4. **Underused white papers** — valuable technical research buried in PDFs

---

## RFQ Repository — Known Scope (Brad's notes)

Brad described the RFQ Repository as the most complex project. His stream-of-consciousness notes:
- Raw material prices
- Pricing / quote templates
- Research / dosage / application templates
- Product charts / graphs / visualizations
- Strain data information
- Export to PDF — internal "dossier" and external proposal format
- Data source integrations (NetSuite, HubSpot, Excel, PDF white papers)

> **Do not quote this project until after a dedicated scoping session.**

---

## Data Sources & Integration Plan

| Source | Type | Phase |
|--------|------|-------|
| Anthropic API | AI/LLM | Live |
| PDF white papers | Static files | Phase 1 (Ask the Docs) |
| FAQ document (Brad's) | Word doc → PDF | Phase 1 (Wiki) |
| Google Drive | Cloud storage | Phase 2 |
| HubSpot | CRM | Phase 2 (RFQ) |
| NetSuite | ERP | Phase 2 (RFQ) |
| Excel files | Spreadsheets | Phase 2 (RFQ) |

> Data sources do not *need* to be integrated per Brad — but the option exists for Phase 2.

---

## Pricing & Commercial

### Phase 1 Quote (active)

| Project | Phase | Range |
|---------|-------|-------|
| Tech Team Wiki | Phase 1 — Core build | $1,500 – $2,000 |
| Tech Team Wiki | Phase 2 — Enhancements | $800 – $1,200 |
| Tech Team Wiki | Phase 3 — AI layer | $600 – $900 |
| Tech Team Wiki | Monthly retainer | $200 – $350/month |
| White Paper AI Chat | Phase 1 — Core build | $1,200 – $1,800 |
| White Paper AI Chat | Phase 2 — Document library | $900 – $1,400 |
| White Paper AI Chat | Phase 3 — Integrations | $800 – $1,200 |
| White Paper AI Chat | Monthly retainer | $150 – $250/month + API costs |

**Suggested opening position:** $3,000 flat for both Phase 1s combined.

### Commercial terms
- 50% deposit to begin, 50% on delivery per project
- Anthropic API costs are pass-through — MicroSynergies owns their own API key and is billed at cost
- Quote valid 30 days from issue date

---

## Privacy & Data Handling

- Documents uploaded to the Anthropic API are retained for 7 days maximum and never used for model training (standard API terms)
- Zero Data Retention (ZDR) is available via Anthropic enterprise contract — worth raising when RFQ Repository is scoped, as it may involve sensitive pricing data
- MicroSynergies should own their own Anthropic API key for production deployment
- All DataForge work uses the standard Anthropic API (not claude.ai consumer interface) — appropriate for confidential client documents

---

## Proposed Engagement Timeline

| Week | Activity |
|------|----------|
| OOO | Brad + TC at conference — no meetings |
| Week 1 (return) | Discovery & kickoff — FAQ doc review, Dale status, stack alignment |
| Weeks 2–3 | Tech Team Wiki Phase 1 build |
| Weeks 4–5 | White Paper AI Chat Phase 1 deployment |
| Week 6 | Budget review + RFQ Repository scoping session |
| Week 7+ | RFQ Repository Phase 2 (TBD scope & budget) |

---

## Deliverables Produced (to date)

| Deliverable | File | Notes |
|-------------|------|-------|
| Interactive demo mockup | `MicroSynergies_Demo_v2.html` | Self-contained HTML, all 4 modules |
| Proposal doc | `MicroSynergies_Proposal_v2.docx` | One-pager, branded |
| Quote doc | `MicroSynergies_Quote_v1.docx` | Two pages: client quote + private hourly reference |
| Ask the Docs app | `ask-the-docs/` repo | Working Node.js app |
