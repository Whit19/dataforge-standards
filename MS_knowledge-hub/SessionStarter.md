# SessionStarter — knowledge-hub
Last updated: 2026-06-09 (end of session 2)

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

**Ask the Docs — Phase 1 built and deployed to Railway.** Basic Auth added. Custom domain `hub.microsynergies.com` configured — awaiting CNAME setup by MicroSynergies IT. Brad has instructions to create Anthropic API key and set $50 usage cap.

**Tech Team Wiki — Pre-build.** Kickoff meeting held 2026-06-10 with Brad and Dale. Follow-up session needed to capture decisions from that meeting and begin build.

**Repo consolidated** — `ask-the-docs` and `tech-team-wiki` merged into single `knowledge-hub` repo.

---

## Next Priorities

1. **Capture kickoff decisions** — hosting, auth, Dale's progress, OneDrive PDF library, wiki stack
2. **MicroSynergies IT** — confirm CNAME record added for `hub.microsynergies.com`
3. **Brad's Anthropic API key** — confirm created and sent securely
4. **Begin wiki Phase 1 build** — scaffold structure, populate content from FAQ doc
5. **Test Ask the Docs on Railway** — upload a white paper, confirm end-to-end flow

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
- [x] Kickoff meeting held with Brad and Dale (2026-06-10) — decisions TBC next session

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
- KH-002 — Dale's wiki progress — to be confirmed next session
- KH-003 — Wiki hosting and stack — to be confirmed next session
- KH-004 — Auth requirement — to be confirmed next session
- KH-005 — Quote scope bullet needs correction before sending
- KH-006 — CNAME record pending — MicroSynergies IT needs to add DNS record
- KH-007 — Brad's Anthropic API key — not yet received

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
| Brad | CTO | Primary champion. Decision-maker. |
| Dale | Tech team | Wiki is his 2026 incentive goal — collaborate, don't replace |
| TC | Leadership | Budget approvals |
| Roy | Operations / Sourcing | RFQ process sign-off |

---

## Reference Docs
- Client context → MicroSynergies.md (company repo)
- Architecture → TechnicalArchitecture.md
- Roadmap → ProjectRoadmap.md
