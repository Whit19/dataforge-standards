# SessionStarter — knowledge-hub
Last updated: 2026-06-30 (end of session 3)

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

**Inactive — client opted to build in-house.** Brad (CTO) replied on 2026-06-30 declining to move forward with the proposal. After reviewing the demos (Wiki + Ask the Docs), MicroSynergies determined they are farther along internally than initially thought and want to give their own team the opportunity to build these capabilities over the coming months. This was a build-vs-buy decision, not a budget rejection — the demo set their internal bar for "what good looks like."

Brad explicitly left the relationship open:
- Will reach back out if internal progress stalls or another project is a better fit for outside collaboration
- Offered to make introductions to other companies who could benefit from DataForge's approach

No further build work is planned unless MicroSynergies re-engages. See DecisionLog.md MS-007 for full decision record.

---

## Next Priorities

None — project inactive pending client re-engagement or referral follow-through. No action items at this time.

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
