# SessionStarter — knowledge-hub
Last updated: 2026-07-23 (end of session 3)

---

## What This Is

The MicroSynergies Knowledge Hub — a unified platform with two modules:

| Module | Purpose |
|--------|---------|
| **Ask the Docs** | AI chat interface over uploaded PDF white papers and supplier docs |
| **Tech Team Wiki** | Human-curated, browsable knowledge base for the T3 tech team |

Both modules share one Express server, one repo, one Railway deployment.

---

## Current Status

**Project status: Inactive — client building in-house.** Brad declined the proposal after seeing the Ask the Docs and Wiki demos, opting to build these capabilities internally at MicroSynergies rather than engage DataForge. Two re-engagement paths remain open: (1) if MicroSynergies' internal build stalls, or (2) via referral introductions Brad offered to other companies.

**Railway service deleted (2026-07-23).** The `knowledge-hub` Railway service (Ask the Docs + Wiki demo, custom domain `hub.microsynergies.com`) has been removed so MicroSynergies can no longer access the demo. The GitHub repo and all code remain intact — a new Railway service can be spun up from the repo at any time if the engagement resumes. Env vars (`ANTHROPIC_API_KEY`, `BASIC_AUTH_USER`/`BASIC_AUTH_PASS`, `GITHUB_TOKEN`, etc.) and the SQLite persistent volume were not preserved and would need to be re-created on redeploy.

KH-006 (CNAME record) and KH-007 (Brad's API key) remain open in the tracker rather than resolved, since they are moot pending any re-engagement rather than genuinely closed.

---

## Next Priorities

1. **Monitor for re-engagement signals** — internal build stalling, or referral leads from Brad materializing
2. **If re-engaged:** re-deploy Railway service from existing repo, re-create env vars, re-confirm CNAME with MicroSynergies IT
3. **Develop referral pipeline** — follow up on Brad's offer to introduce DataForge to other companies

---

## Recent Completed

- [x] Ask the Docs — full Phase 1 Express app (upload, ask, delete session)
- [x] Files API 404 bug fixed — `betas` param inside SDK call (not header override)
- [x] Logo fixed — base64 data URI in `<img>` tag
- [x] MicroSynergies branding + DataForge footer
- [x] Brad's FAQ doc reviewed in full — content structure decided
- [x] Wiki demo v4 built with real FAQ content (5 categories, 12 pages)
- [x] Repos consolidated into single `knowledge-hub` repo
- [x] Basic Auth added to server.js (BASIC_AUTH_USER / BASIC_AUTH_PASS env vars)
- [x] Railway deployment configured — env vars set, custom domain added
- [x] GitHub personal access token approach documented for wiki commits
- [x] Brad's API key setup instructions prepared
- [x] Kickoff meeting held with Brad and Dale (2026-06-10)
- [x] Merged Technical Architecture doc produced (covers both modules, MS365/OneDrive corrected)
- [x] Quote sent to Brad; Brad's reply received and reviewed (2026-06-30)
- [x] Reply drafted and sent to Brad acknowledging the decision, offering ad-hoc help, thanking him for referral interest

---

## Wiki Content Structure (decided)

| Category | Pages |
|----------|-------|
| Sales | Sales Process, TEQs by Industry, Pricing & Quotes |
| Products & Capabilities | Certifications & Caps, Toll Blending & Campaigns, Samples & Shelf Life |
| Regulatory & Markets | International Shipping, EPA & Organic |
| Systems | HubSpot, NetSuite |
| Content & Tools | Content Creation, NDAs & Payment Terms |

---

## Open Issues
> See IssuesTracker.md

- KH-001 — In-memory sessions reset on server restart (deferred to Phase 2)
- KH-002 — Dale's wiki progress — moot, project inactive
- KH-003 — Wiki hosting and stack — moot, project inactive
- KH-004 — Auth requirement — moot, project inactive
- KH-005 — Quote scope bullet needs correction before sending
- KH-006 — CNAME record pending — left open per Tom's preference
- KH-007 — Brad's Anthropic API key — left open per Tom's preference

---

## Environment

```
Runtime:    Node.js (v22+)
Port:       process.env.PORT || 3000
API key:    ANTHROPIC_API_KEY [Railway env var]
Auth:       BASIC_AUTH_USER / BASIC_AUTH_PASS [Railway env var]
GitHub:     GITHUB_TOKEN / GITHUB_REPO_OWNER / GITHUB_REPO_NAME [Railway env var]
Model:      claude-sonnet-4-20250514
Hosting:    Railway — hub.microsynergies.com (CNAME pending)
```

---

## Quick Start

```bash
cd knowledge-hub
npm install
# Create .env directly (do not cp .env.example):
# ANTHROPIC_API_KEY=sk-ant-...
# PORT=3000
npm start
# → http://localhost:3000
```

---

## Key Contacts

| Name | Role | Notes |
|------|------|-------|
| Brad | CTO | Primary champion. Decided to build in-house (2026-06-30). Open to future re-engagement or referrals. |
| Dale | Tech team | Wiki was his 2026 incentive goal — now an internal build |
| TC | Leadership | Budget approvals |
| Roy | Operations / Sourcing | RFQ process sign-off |

---

## Reference Docs
- Client context → MicroSynergies.md (company repo)
- Architecture → TechnicalArchitecture.md
- Roadmap → ProjectRoadmap.md
