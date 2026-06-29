# HQ Dashboard — Decision Log

**Last Updated:** March 2026  
**Purpose:** Track key architectural, design, and product decisions with rationale.  
This document helps future sessions avoid relitigating settled questions and understand *why* things were built the way they were.

---

## How to Use This Log

Each entry has:
- **Decision:** What was decided
- **Rationale:** Why this choice was made
- **Alternatives Considered:** What else was on the table
- **Status:** Active | Revisit if... | Superseded

---

## Architecture Decisions

---

### AD-01 — Claude API Calls Live in GAS, Not the Browser

**Decision:** The Anthropic Claude API is called from Google Apps Script (server-side), never from browser JavaScript.

**Rationale:** The Anthropic API does not support CORS, so direct `fetch()` calls from a browser will always fail with a CORS error. GAS's `UrlFetchApp` has no such restriction and runs server-side.

**Alternatives Considered:**
- Browser direct call — blocked by CORS, not viable
- Proxy server (e.g. Cloudflare Worker) — adds another dependency and cost; GAS is free and already in use

**Status:** Active

---

### AD-02 — Calendar Events Sent via GET Param, Not POST

**Decision:** Calendar events from the browser are encoded as compact JSON and sent as a `?cal=` query parameter on GET requests to GAS, rather than via POST.

**Rationale:** POST requests from the browser to GAS Web Apps require CORS preflight, which GAS does not support properly — POST consistently fails from browsers that enforce CORS strictly. GET requests work reliably with the proxy fallback.

**Alternatives Considered:**
- POST endpoint — implemented but abandoned due to CORS failures
- Server-Sent Events — overkill
- Store in localStorage and include in next generate call — similar to current approach but can't exceed URL limits; chosen solution caps at 60 events / 4000 chars

**Status:** Active. Revisit if GAS adds CORS support for POST.

---

### AD-03 — Google Sheets as the Data Store

**Decision:** All persistent data (emails, summaries, calendar events) is stored in Google Sheets tabs, not a database.

**Rationale:** GAS has native, fast read/write access to Sheets. No external database needed. Sheets also provides a human-readable audit log the user can view directly. Zero cost.

**Alternatives Considered:**
- Firebase Realtime DB — adds cost and complexity
- GAS Properties Store — too small for email history
- External Postgres/Supabase — major added complexity, cost, auth

**Status:** Active

---

### AD-04 — Single HTML File, No Build Tools

**Decision:** The dashboard is a single `.html` file with embedded CSS and JS. No npm, no bundler, no framework.

**Rationale:** Simplicity and portability. A single file can be opened locally, hosted on Netlify Drop, or saved anywhere. No build step = no broken dependencies between sessions. Easy to version and share.

**Alternatives Considered:**
- React SPA — would require build tooling; breaks the single-file portability
- Vue/Svelte — same problem
- Separate CSS/JS files — adds hosting complexity, still no build benefit at this scale

**Status:** Active. Revisit if the file exceeds ~3000 lines and becomes hard to maintain.

---

### AD-05 — CORS Proxy Waterfall for ICS Fetching

**Decision:** Calendar .ics feeds are fetched via a waterfall of three public CORS proxies: allorigins.win → corsproxy.io → codetabs.com. First success wins.

**Rationale:** ICS feeds are served from external domains (TeamSnap, LeagueApps, etc.) with no CORS headers. Direct browser fetch fails. Multiple proxies provide resilience when one is rate-limited or down.

**Alternatives Considered:**
- Route .ics fetching through GAS — would work, but adds a round trip and complicates the GAS endpoint
- Ask user to self-host a proxy — too much friction
- Single proxy — less resilient; proxies go down periodically

**Status:** Active. Revisit if proxies become unreliable; GAS route is the backup plan.

---

### AD-06 — Settings Stored in localStorage

**Decision:** The GAS Web App URL and team calendar URLs are saved in the browser's `localStorage`.

**Rationale:** No backend auth system. Users configure once; settings persist across page loads. Simple.

**Alternatives Considered:**
- Hard-code in HTML — requires users to edit code
- Store in Google Sheet via GAS — chicken-and-egg (need the URL to store the URL)
- URL hash params — not private, can't store multiple calendars cleanly

**Status:** Active. Revisit when building multi-device sync or kid profiles.

---

### AD-07 — GAS Web App Access: "Anyone" (No Auth)

**Decision:** The GAS Web App is deployed with "Access: Anyone" — no login required to call the endpoint.

**Rationale:** The dashboard is a personal tool. OAuth would add significant friction. The Web App URL functions as an unguessable token (security through obscurity is acceptable for this personal use case).

**Alternatives Considered:**
- OAuth via Google Sign-In — too much friction for a personal tool
- API key in request headers — GAS Web Apps don't support custom auth headers reliably
- Password in query param — functional but messy

**Status:** Active. Revisit if the app becomes multi-user or shared publicly.

---

### AD-08 — Claude Model: claude-sonnet-4-5

**Decision:** The GAS script uses `claude-sonnet-4-5` as the model for briefing generation.

**Rationale:** Strong reasoning capability for parsing unstructured email text, good JSON output reliability, reasonable cost for daily use (~1 call/day).

**Alternatives Considered:**
- claude-haiku — faster/cheaper but less reliable JSON and weaker summarization
- claude-opus — more capable but significantly more expensive for daily auto-runs

**Status:** Active. Update model string when newer versions are available.

---

### AD-09 — Briefing Uses Full 15-Day Email History (Not Just Latest Scan)

**Decision:** When generating a briefing, `generateSummaryWithClaude()` reads the full `EmailHistory` sheet (15 days) and merges it with the current scan results, rather than only using emails from the latest scan.

**Rationale:** Initial implementation only passed the current scan's emails. This caused Claude to miss context from earlier in the week (e.g., a game confirmed 10 days ago wouldn't show up). The rolling history sheet ensures all relevant context is included.

**Alternatives Considered:**
- Just use the latest scan — simpler, but misses history
- Only pass emails from the last 7 days — reasonable compromise; current approach passes all 15 days which is slightly more than needed but safe

**Status:** Active

---

## Design Decisions

---

### DD-01 — Design System: Bebas Neue + DM Sans + DM Mono

**Decision:** Typography stack is Bebas Neue (display/headers), DM Sans (body), DM Mono (labels/codes/meta).

**Rationale:** Bebas Neue gives an athletic, energetic feel appropriate for a sports app. DM Sans is highly readable at small sizes on mobile. DM Mono provides clear visual hierarchy for metadata like dates, tags, and app labels.

**Status:** Active

---

### DD-02 — Mobile-First but Desktop-Capable

**Decision:** Responsive layout that prioritizes mobile viewport but works well on desktop with max-width 1120px container.

**Rationale:** Parents check this on their phone. Desktop is secondary.

**Status:** Active

---

### DD-03 — Briefing Renders Inline in Summary Tab (Not Modal/Overlay)

**Decision:** The briefing output (priority, cards, timeline, actions) renders directly in the Summary tab, replacing the empty state. No separate "output" page or modal.

**Rationale:** Fewer taps. User opens app → sees briefing immediately. "Refresh" regenerates in place.

**Alternatives Considered:**
- Separate output route/page — navigation overhead
- Modal overlay — hard to scroll on mobile, loses context

**Status:** Active

---

## Product Decisions

---

### PD-01 — Two Separate Dashboards (Sports HQ + School HQ)

**Decision:** Sports and school activities get separate dashboards and separate GAS backends, not a single merged system.

**Rationale:** The mental models are different. Sports = upcoming games, practices, coach messages. School = assignments, tests, deadlines, teacher communication. Keeping them separate means each can be optimized for its use case without compromise. They share a design system but are functionally independent.

**Alternatives Considered:**
- Single merged dashboard — simpler, but the prompt engineering and card design would be awkward for both use cases
- Single GAS with routing — possible, but harder to maintain

**Status:** Active. May revisit with a unified shell navigation layer in Phase 3c.

---

### PD-02 — Manual Updates Tab as Fallback Input

**Decision:** Include a tab where users can paste text directly from any sports app as a fallback when email scanning misses something.

**Rationale:** Not everything comes via email. Some coaches use only app-native messaging. Manual paste ensures the briefing stays useful even with incomplete email coverage.

**Status:** Active. Note: manual updates are currently NOT being passed to Claude's prompt. This is a known gap (see Issues Tracker).

---

## Session Notes

> Add entries when decisions are made, revisited, or reversed.

- **Session 1 (Mar 2026):** Log initialized. Decisions AD-01 through AD-09, DD-01 through DD-03, PD-01 through PD-02 documented from MVP state.

