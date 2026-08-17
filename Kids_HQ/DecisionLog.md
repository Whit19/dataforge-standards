# HQ Dashboard — Decision Log

**Last Updated:** August 2026  
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

**Decision:** All persistent data (emails, summaries, calendar events, settings) is stored in Google Sheets tabs, not a database.

**Rationale:** GAS has native, fast read/write access to Sheets. No external database needed. Sheets also provides a human-readable audit log the user can view directly. Zero cost.

**Alternatives Considered:**
- Firebase Realtime DB — adds cost and complexity
- GAS Properties Store — too small for email history
- External Postgres/Supabase — major added complexity, cost, auth

**Status:** Active

---

### AD-04 — Single HTML File, No Build Tools

**Decision:** The dashboard is a single `.html` file with embedded CSS and JS. No npm, no bundler, no framework.

**Rationale:** Simplicity and portability. A single file can be opened locally, hosted on GitHub Pages, or saved anywhere. No build step = no broken dependencies between sessions. Easy to version and share.

**Alternatives Considered:**
- React SPA — would require build tooling; breaks the single-file portability
- Vue/Svelte — same problem
- Separate CSS/JS files — adds hosting complexity, still no build benefit at this scale

**Status:** Active. Revisit if the file exceeds ~3000 lines and becomes hard to maintain.

---

### AD-05 — CORS Proxy Waterfall for ICS Fetching

**Decision:** Calendar .ics feeds are fetched via a waterfall of three public CORS proxies: allorigins.win → corsproxy.io → codetabs.com. First success wins.

**Rationale:** ICS feeds are served from external domains (TeamSnap, LeagueApps, etc.) with no CORS headers. Direct browser fetch fails. Multiple proxies provide resilience when one is rate-limited or down.

**Status:** **Superseded by AD-10.** In practice this became the dashboard's single biggest reliability problem (see ISSUE-002) — kept here for history.

---

### AD-06 — Settings Stored in localStorage

**Decision:** The GAS Web App URL and team calendar URLs are saved in the browser's `localStorage`.

**Status:** **Superseded by AD-11.** localStorage is still used (and is still the only place the GAS URL itself lives), but Children/Activities/Calendars/senders now also sync to the Sheet — see AD-11.

---

### AD-07 — GAS Web App Access: "Anyone" (No Auth)

**Decision:** The GAS Web App is deployed with "Access: Anyone" — no login required to call the endpoint.

**Rationale:** The dashboard is a personal tool. OAuth would add significant friction. The Web App URL functions as an unguessable token (security through obscurity is acceptable for this personal use case).

**Alternatives Considered:**
- OAuth via Google Sign-In — too much friction for a personal tool
- API key in request headers — GAS Web Apps don't support custom auth headers reliably
- Password in query param — functional but messy

**Status:** Active. Revisit if the app becomes multi-user or shared publicly. **Important knock-on effect (AD-11):** because this endpoint has no auth, the Settings sheet it reads/writes is world-readable by anyone with the URL — this is why the Anthropic API key must never be allowed into that sheet (see AD-11).

---

### AD-08 — Claude Model: claude-sonnet-4-5

**Decision:** The GAS script uses `claude-sonnet-4-5` as the model for briefing generation.

**Rationale:** Strong reasoning capability for parsing unstructured email text, good JSON output reliability, reasonable cost for daily use (~1 call/day).

**Alternatives Considered:**
- claude-haiku — faster/cheaper but less reliable JSON and weaker summarization
- claude-opus — more capable but significantly more expensive for daily auto-runs

**Status:** Active. Update model string when newer versions are available.

---

### AD-09 — Briefing Uses Full Rolling Email History (Not Just Latest Scan)

**Decision:** When generating a briefing, `generateSummaryWithClaude()` reads the full per-category `EmailHistory`/`SchoolEmailHistory` sheet and merges it with the current scan results, rather than only using emails from the latest scan.

**Rationale:** Initial implementation only passed the current scan's emails. This caused Claude to miss context from earlier in the week (e.g., a game confirmed 10 days ago wouldn't show up). The rolling history sheet ensures all relevant context is included.

**Status:** Active. See AD-14 for a bug that was hiding in this rolling-history merge.

---

### AD-10 — ICS Calendar Fetching Moved Server-Side Through GAS

**Decision:** Calendar `.ics` feeds are now fetched inside Google Apps Script via `UrlFetchApp` (`fetchIcsServerSide()`, exposed as `doGet(?action=ics&url=...)`), not through the three-proxy browser waterfall from AD-05.

**Rationale:** The proxy waterfall from AD-05 broke in production — diagnosed directly (not guessed): a live fetch test against each proxy showed allorigins.win timing out, and corsproxy.io had changed its API to no longer proxy the old query format, returning its own marketing homepage instead of the calendar data. `UrlFetchApp` has no CORS restriction at all and doesn't depend on a third party's uptime or API stability for something this core to the app.

**Alternatives Considered:**
- Try more/different public CORS proxies — same underlying reliability problem, just deferred
- Ask the user to self-host a proxy — too much friction for a personal tool

**Status:** Active. Note: `allorigins.win` still appears once, as a fallback in `fetchFromGas()` purely for reaching the GAS Web App itself if a direct fetch to `/exec` fails — that's unrelated to calendar fetching and should not be confused with the old ICS waterfall.

---

### AD-11 — Settings Also Synced to a GAS-Backed Settings Sheet

**Decision:** Children/Activities/Calendars/senders/maxEmails now sync to a "Settings" tab in the Google Sheet via `action=saveSettings` / `action=getSettings` GET endpoints, in addition to `localStorage`. The GAS Web App URL itself stays localStorage-only (it's the thing you need in order to reach the sheet in the first place).

**Rationale:** localStorage doesn't sync across browsers or devices, and the user needed settings to follow them without adding real authentication. A "Load Settings from Cloud" button plus an auto-pull heuristic (fresh browser + no local settings + a GAS URL already pasted in) covers the common case without prompting on every load.

**Alternatives Considered:**
- Export/import a settings JSON file — too manual for routine use
- A real backend database — same rationale against as AD-03

**Critical security constraint:** `ANTHROPIC_API_KEY` must NEVER be included in the settings object that round-trips through `saveSettings`/`getSettings`. Because of AD-07 (Access: Anyone, no auth), that Settings sheet is readable by anyone who has the Web App URL — this app's whole security model is "the URL is the password." `saveSettings()` explicitly does `delete settingsObj.ANTHROPIC_API_KEY; delete settingsObj.apiKey;` before writing, and the key remains a hardcoded script-level constant in `SportsEmailScanner.gs` only. **Do not remove this delete, and do not add the key to any object that gets sent to `saveSettings`.**

**Status:** Active.

---

### AD-12 — Child → Activity → Sender/Calendar Linking Model (Not Flat Category Tags)

**Decision:** Settings is structured as Children (id, name, color) → Activities (id, category `sports`/`school`, name, one-or-more linked `childIds`, `daysBack`/`daysForward`, list of email-sender matches) → Calendars (each linked to exactly one Activity via `activityId`). Gmail-scanned emails and ICS calendar events both carry that `activityId` all the way through the Sheets and JSON responses.

**Rationale:** Previously, senders and calendars only carried a flat `sports`/`school` category tag — there was no way to know *which kid* a given email or calendar event belonged to, so nothing could be filtered per child even with multiple kids configured in the same dashboard.

**Alternatives Considered:**
- Tag senders/calendars directly with `childIds`, skipping the Activity layer — rejected because a team or school is the real-world unit siblings actually share (one team calendar can cover two kids); Activity is the natural join point, not the raw sender.

**Migration:** `migrateLegacySettings()` (GAS) and `migrateLegacyLocal()` (dashboard JS) auto-detect the old flat shape and wrap it into an "(unsorted)" default Activity per category with no children assigned, so nothing already configured breaks on upgrade — the user re-sorts senders/calendars onto real children afterward, at their own pace.

**Status:** Active.

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

**Status:** Active

---

## Product Decisions

---

### PD-01 — Two Separate Dashboards (Sports HQ + School HQ)

**Status:** **Superseded by PD-03.** School HQ was ultimately built inside the same dashboard, not as a separate system. Kept here for history.

---

### PD-02 — Manual Updates Tab as Fallback Input

**Decision:** Include a tab where users can paste text directly from any sports app as a fallback when email scanning misses something.

**Rationale:** Not everything comes via email. Some coaches use only app-native messaging. Manual paste ensures the briefing stays useful even with incomplete email coverage.

**Status:** Active, but see ISSUE-001 — manual updates are still not being passed to Claude's prompt, and the tab's hardcoded 7-app list (TeamSnap/SportsEngine/GameChanger/GroupMe/Playmetrics/League/Carpool Kids) now predates the user-editable Activity/sender model from AD-12, so it no longer necessarily matches what the user has actually configured. Worth reworking both gaps together rather than separately.

---

### PD-03 — Unified Single Dashboard Instead of Separate Sports HQ / School HQ

**Decision:** Reversing PD-01 — School HQ was built as a second category (`sports`/`school`) inside the *same* `sports-dashboard.html` file and the *same* `SportsEmailScanner.gs` backend, writing to the *same* spreadsheet, just with separate per-category sheet tabs (`SchoolEmails`/`SchoolEmailHistory`/`SchoolSummary`/`SchoolCalendarEvents` alongside the original Sports tabs).

**Rationale:** User's own call, made explicitly before School HQ work started: building School as a fully separate system first (per the original Phase 3 roadmap) risked throwing away layout/view work once School was added, since the two would likely need to change together. The user also wanted Settings cleaned up in the same pass (moving hardcoded senders/date-windows into an editable UI), which only makes sense with one shared Settings model rather than two.

**Alternatives Considered:**
- Fully separate School dashboard/script/sheet as originally scoped (see old Phase 3 in Project Roadmap)
- Shared design system but independent backends — rejected for the same Settings-unification reason

**Status:** Active. ProjectRoadmap.md's Phase 3 has been marked complete-with-pivot to reflect this.

---

### PD-04 — Global Child Filter Pill Row (Calendars + Email History, later extended to Summary)

**Decision:** Added a "👪 All / [Child name]" pill row above the existing Sports/School switch. It was initially scoped to filter the Calendars and Email History tabs only — the Summary/briefing tab was deliberately left unfiltered at first, since AI-generated cards had no structured per-child tag to filter on.

**Follow-up (same phase):** the user reported this as "not working" for Summary rather than accepting it as a known limitation — a fair reaction, since a filter pill that visibly does nothing on one whole tab reads as broken. Rather than re-explain the limitation, Claude's requested JSON schema was extended: cards, timeline entries, and actions each now get a `childIds: [...]` field, populated using a `KNOWN CHILDREN (id = name)` legend fed into the prompt. Items with no tag (general/unclear) default to showing under every filter instead of disappearing, so an AI miss degrades gracefully rather than silently hiding content.

**Status:** Active. Because the tagging comes from the model's own output rather than deterministic code, treat it as "monitoring" quality, not guaranteed — see ISSUE-010.

---

### AD-13 — Inline HTML Event-Handler Attributes Must Avoid Names That Collide With `document`/`window` Built-ins

**Decision / standing rule:** Never name a global JS variable or function `children`, `removeChild`, `name`, `open`, `close`, `history`, `top`, `self`, `location`, `status`, `find`, or similar **when it will be referenced as a bare identifier inside an inline `onclick=`/`oninput=`/`onchange=""` attribute string.**

**Root cause (confirmed, not guessed):** per the HTML spec, inline event-handler attributes execute inside an implicit `with(document){ with(form){ with(element){ ... } } }` scope. `document.children` and `document.removeChild` are real built-in properties — they silently shadow a same-named page-level global. Assignments/calls inside the handler then silently operate on the DOM instead of app state, usually with no visible error for index 0 and a swallowed exception for anything past it. This was confirmed with an isolated two-line Playwright repro *before* any fix was written, per this project's standing debugging preference of finding the actual cause rather than assuming.

**Symptom it caused:** typing a child's name in Settings appeared to work (the browser input showed the typed text) but never persisted — `children[i].name = ...` in the inline handler was actually setting a property on `document.children[i]` (an `<html>` element or `undefined`), not on the app's real `children` array. The delete (✕) button silently no-op'd or threw for the same reason (`removeChild` resolved to the native `Node.prototype.removeChild`).

**Fix applied:** renamed the global array to `kids` and the delete function to `deleteChild()` throughout `sports-dashboard.html`; audited every other inline handler in the file for the same class of collision (none further found).

**Status:** Active — recorded here so a future session adding new Settings fields doesn't reintroduce this. Prefer referencing state via a wrapper function call (`onclick="doSomething(...)"`) over bare array/object indexing directly inside inline attributes where practical, since the wrapper function itself isn't executed inside the `with()` scope.

---

### AD-14 — Rolling History Sheet Merge Must Let Fresh Scan Data Overwrite Stale Rows

**Decision / standing rule:** `updateHistorySheet()` merges by `date|subject` key and lets the *newest* scan's row win on a collision — never keep the first-seen row's data forever.

**Root cause (confirmed by reading the merge logic, not guessed):** `scanEmails()` already fully re-derives the correct `App`/`ActivityId` for every email in the current `daysBack` window on *every* run — it is not incremental. The bug was that `updateHistorySheet()`'s merge only ever appended rows for genuinely-new `date|subject` keys and left existing rows completely untouched. Since most emails in the current window were already present in the sheet from a prior run, they kept whatever `ActivityId` they were first tagged with — even a blank one from before the user had linked that sender to a child — forever, regardless of how many times Settings was reorganized afterward.

**Symptom it caused:** per-child email filtering appeared to "work for some kids, not others" — new emails scanned after a Settings reorg got the correct fresh tag, but anything already sitting in history from before the reorg kept its stale tag indefinitely.

**Fix applied:** merge now builds a `Map` keyed by `date|subject`, seeds it from existing rows, then overwrites with the current scan's rows — fresh data always wins.

**Status:** Active. Note: this only heals rows within the *current* `daysBack` scan window on the next scan — anything a sender's `daysBack` window doesn't reach won't be corrected until it's re-scanned or ages out.

---

## Session Notes

> Add entries when decisions are made, revisited, or reversed.

- **Session 1 (Mar 2026):** Log initialized. Decisions AD-01 through AD-09, DD-01 through DD-03, PD-01 through PD-02 documented from MVP state.
- **Session — 2026-08-17:** School HQ delivered as a Unified dashboard (PD-03, superseding PD-01). Settings rebuilt around a Child → Activity linking model (AD-12) with cloud sync (AD-11, superseding AD-06). ICS fetching moved server-side through GAS (AD-10, superseding AD-05), which also fixed ISSUE-002. Added a global Child filter (PD-04), later extended to the Summary tab via structured `childIds` tagging in Claude's output. Two real bugs found and fixed with root-caused, test-verified diagnoses: a `document`-shadowing bug blocking Add/Edit/Delete Child (AD-13), and a stale-tag bug in the rolling email history merge (AD-14). See IssuesTracker.md for the corresponding closed issues.
