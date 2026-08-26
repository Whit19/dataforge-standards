# HQ Dashboard — Decision Log

**Last Updated:** 2026-08-19  
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

**Status:** **Superseded by DD-04** (2026-08-26) — Quicksand/Nunito Sans replaced this typography system project-wide, following feedback that it read "too digital." Kept here for history.

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

### PD-05 — Single Category → Child → Activity Filter Cascade (Replaces Separate Per-Sender / Per-Calendar Filters)

**Decision:** Calendars and Email History previously had their own separate filter rows — Calendars filtered by calendar name (`calName`), Email History by individual sender/app label. Both are now driven by one shared three-tier cascade: Category (Sports/School) → Child → Activity, sitting above the tabs as global pill rows.

**Rationale:** The per-sender chip row in Email History grew to 20+ chips as more senders were configured — unusable. The per-calendar-name row in Calendars had the same problem at smaller scale. Activity (a team or school) is the real unit both senders and calendars already roll up to via `activityId` — filtering by Activity directly is both more useful and inherently bounded (as many chips as configured Activities, not as many as configured senders).

**Alternatives Considered:**
- Keep per-sender/per-calendar filters but add a "show more" collapse — rejected, still two separate mental models for functionally the same filter
- Group by sender app label instead of Activity — rejected, doesn't solve the underlying "which team/school" question the user actually has

**Trade-off accepted:** Switching Category or Child now resets the Activity filter back to "All" rather than trying to preserve a selection across incompatible option sets — simplest, least-surprising default.

**Status:** Active.

---

### AD-15 — Global Scan/Briefing Window Instead of Per-Activity

**Decision:** `daysBack` / `daysForward` moved from a field on each Activity to one global setting (`Scan & Briefing Window` in Settings), applied uniformly across every Activity in both categories.

**Rationale:** There was never a real use case for different scan windows per team — it was a leftover from before Activities existed, and it was one more field to fill in on every single Activity card, adding to Settings page sprawl.

**Migration:** On first load after this change, if old per-activity values are present without a new top-level `daysBack`/`daysForward`, the widest of the old values is carried forward automatically as the new global default — nothing narrows silently on upgrade. Implemented once in both GAS `migrateLegacySettings()` and the dashboard's `migrateLegacyLocal()`.

**Status:** Active.

---

### AD-16 — Explicit `todayISO` + "No Stale Content" Prompt Rule, Plus a `dateISO` Field for Deterministic Filtering

**Decision:** Both prompt builders (`buildSportsPrompt`, `buildSchoolPrompt`) now receive `todayISO` explicitly and an instruction to never surface a card/timeline/action item whose date has already passed relative to it. Timeline and action items now also carry a `dateISO` field (YYYY-MM-DD), used by the dashboard to (a) drop any item with a past date client-side regardless of what the model returned, and (b) group the 7-Day Schedule by actual day.

**Rationale:** The app intentionally feeds Claude the full rolling email history for context (AD-09/BM-14 — nothing gets missed), which means old emails describing since-passed events are always present. Without an explicit rule and a deterministic backstop, that intentional design surfaced stale information as if current (ISSUE-011) — confirmed directly: a briefing generated Aug 17 included a volunteer signup for an event that happened Aug 15.

**Alternatives Considered:**
- Trim the email history window fed to Claude — rejected, defeats the purpose of AD-09/BM-14 (missing real context is worse than including old context, as long as staleness is handled)
- Rely on the prompt instruction alone, no client-side backstop — rejected, consistent with this project's established practice (see ISSUE-010/PD-04) of not fully trusting model instruction-following for anything filterable deterministically

**Status:** Active. See also BestMethods BM-20.

---

### AD-17 — Calendar Events Now Carry a Real Computed ISO Date, Not Just a Display String

**Decision:** The `?cal=` payload sent from the dashboard to GAS now includes a `di` field (ISO date), sourced directly from the browser's own `Date` object for each loaded calendar event. GAS stores it in a new `DateISO` column on each category's CalendarEvents sheet and surfaces it explicitly in the prompt's CALENDAR EVENTS section as `{dateISO: YYYY-MM-DD}` next to each event. The prompt now instructs Claude to copy that value directly for calendar-sourced items rather than compute it.

**Rationale:** Before this change, calendar events were only described to Claude with a display string like `"Mon, Aug 17"` — no year, no real computed date — requiring Claude to infer the actual calendar date from context. This was unreliable in practice: calendar-derived timeline items (recurring practices, a tee time, a tryout) were landing in an undated "Other" bucket instead of their correct day (ISSUE-013). The fix removes the need for the model to compute something the app already knows for certain.

**Alternatives Considered:**
- Ask Claude more forcefully to compute the date correctly (stronger prompt wording only) — rejected as the primary fix; the underlying problem is unnecessary model computation, not insufficient instruction
- Have the dashboard build the entire timeline for calendar-sourced events deterministically, bypassing Claude for that portion — rejected for this pass, since the current design relies on Claude to dedupe/synthesize calendar and email sources together; worth reconsidering only if dateISO issues persist after this fix

**Status:** Active. See also BestMethods BM-21.

---

### AD-18 — To-Do List Is Persistent But Local-Only (Not Synced to GAS)

**Decision:** The new To-Do tab stores items in `localStorage` only, separate from the AI-regenerated briefing content (which is why it's persistent at all — a briefing regenerates on every Refresh, so checkmarks on the briefing's own Action Items would be wiped out constantly).

**Rationale:** Fastest path to a genuinely useful feature this session; syncing would mean extending the GAS Settings-sheet pattern (AD-11) to a new data type, a larger scope than warranted for a same-session addition.

**Trade-off accepted, deliberately not solved this session:** unlike Settings, the To-Do list does not sync across browsers/devices via GAS. Logged in ProjectRoadmap backlog as a future decision, not an oversight.

**Status:** Active. Revisit if cross-device To-Do access becomes a real need.

---

### AD-19 (operational, not code) — Deployment Must Always Come From `tjunker9@gmail.com`

**Decision/finding:** Apps Script's "Execute as: Me" binds a deployment permanently to whichever Google account was logged in at Deploy time. A deployment made from any account other than `tjunker9@gmail.com` (the Gmail inbox actually being scanned) silently breaks all Gmail operations, with no error until a Gmail-touching action is exercised — see IssuesTracker ISSUE-016 for the full diagnosis, which involved ruling out a stale OAuth token and missing consent before landing on the actual cause.

**Rationale for logging this as a decision, not just a bug fix:** there is no code change that prevents this — it's a standing operational rule that needs to survive across sessions and devices, not something a future fix can eliminate.

**Status:** Active. See also BestMethods BM-19.

---

### AD-20 — GroupMe Messages Normalized Into the Existing Email History Shape

**Decision:** GroupMe messages are normalized into the same 6-column row shape already used for email (`Date, From, App, Subject, Body, ActivityId`) and merged into the same rolling `EmailHistory`/`SchoolEmailHistory` sheet via the existing `updateHistorySheet()` — not a new parallel sheet, tab, or prompt section.

**Rationale:** `updateHistorySheet()` already handles fresh-scan-wins merging by key (AD-14). Reusing it means `generateSummaryWithClaude()` and both prompt builders need zero changes for GroupMe content to flow into the same daily briefing — GroupMe just becomes another row source, tagged `App: 'GroupMe'`.

**Alternatives Considered:**
- A separate `GroupMeHistory` sheet + a second prompt section — rejected as unnecessary duplication of merge/prompt logic for data that's conceptually the same shape ("an update arrived from an app") as email.

**Status:** Active.

---

### AD-21 — Secrets Never Hardcoded as Script Consts; PropertiesService Is the Standard

**Decision:** `ANTHROPIC_API_KEY`, `GROUPME_ACCESS_TOKEN`, and any future script-level secret are read via `PropertiesService.getScriptProperties().getProperty(...)` at runtime, set manually through the Apps Script editor's Project Settings → Script Properties UI — never as a literal string assigned to a const in `SportsEmailScanner.gs`.

**Rationale:** A live Anthropic API key was committed to this file as a plaintext const and pushed to the repo's remote (see IssuesTracker ISSUE-017). BestMethods BM-15 had assumed "the GAS script isn't stored in GitHub" as the safeguard — that assumption didn't hold in practice, since the file is git-tracked (in a private repo, but see BM-23: private isn't a durable secret boundary). PropertiesService removes the secret from any file that could ever be committed, regardless of repo visibility or tracking status.

**Alternatives Considered:**
- Relying on repo privacy alone — rejected; private repos aren't a durable secret boundary (collaborators, an accidental visibility change, or a compromised GitHub account all bypass it)
- A `.gitignored` local secrets file — rejected as unnecessary complexity when Apps Script already provides a built-in, git-invisible property store

**Status:** Active. Corrects the implicit assumption behind BM-15 — see BestMethods for the corrected version.

---

### AD-22 — PWA Service Worker Uses Network-First for the Dashboard, Shell-Only Precache

**Decision:** The new service worker (`sw.js`) does not aggressively cache `sports-dashboard.html`. It precaches only the small PWA shell (manifest + icons) and uses a network-first strategy for navigation requests, falling back to a cached/offline message only if genuinely offline with nothing cached yet.

**Rationale:** The dashboard is a single, frequently-changing HTML file redeployed on every GitHub Pages push (AD-04). An aggressively cached service worker risks showing a stale summary or an outdated UI after a deploy — worse than the app simply requiring a live connection, for something this update-frequent.

**Alternatives Considered:**
- Full offline-first precaching of the dashboard shell — rejected for the staleness risk above
- No service worker at all — rejected; Chrome/Edge require one with a fetch handler for the "Install App" prompt to be eligible at all

**Status:** Active.

---

### PD-06 — Activity Filter Extended to Summary via `activityId` Tagging (Same Pattern as PD-04)

**Decision:** Claude's card/timeline/action schema gains an `activityId` field alongside the existing `childIds`, populated from a new `KNOWN ACTIVITIES (id = name)` legend fed into both prompt builders. The dashboard's `briefingItemMatches()` now checks the active Activity filter against it, and `setActivityFilter()` re-renders Summary — previously it only re-rendered Calendars and Email History.

**Rationale:** Reported directly by the user as broken, not as a known limitation to accept — the Activity pill visibly narrowed Calendars and Email History but had no effect on Summary at all, since there was no `activityId` anywhere in the model's output shape to filter on. Same underlying gap PD-04 solved for the Child filter, just one filter tier later.

**Alternatives Considered:**
- Derive Activity from the card's existing `source` string (e.g. "Coach Emmett") — rejected; sources don't map 1:1 to Activities, and email rows already carry a deterministic `activityId` upstream that the prompt can just pass through instead of asking the model to re-derive it.

**Status:** Active. Same graceful-degradation rule as `childIds` (PD-04): an item with no/unresolved `activityId` shows under every Activity filter rather than disappearing. Treat as "monitoring" quality, not guaranteed, for the same reason as ISSUE-010 — see BestMethods BM-25.

---

### PD-07 — Briefing Card Header Shows "Child — Sport — Source"

**Decision:** Each Summary card's header line now reads `Child — Sport — Source` (e.g. "Whit — MUHS Lax — Coach Emmett") instead of just the source. Any segment Claude didn't tag (no `childIds`, no resolved `activityId`) is omitted rather than shown blank, so a fully-untagged card still degrades to just the source, same as before this change.

**Rationale:** User request, made directly reviewable in a screenshot — with several kids/teams active at once, "Coach Emmett" alone doesn't say which kid or team a card is about without opening it.

**Status:** Active. Depends on PD-06's `activityId` and the existing `childIds` (PD-04) both being present and correctly tagged to show its full form.

---

### PD-08 — "Family Members" Replaces "Children" as a UI Label

**Decision:** Every visible Settings label reading "Children"/"Child" is
relabeled to "Family members"/"Family member" — the section header, helper
text, the "CHILD'S NAME" field label, and the per-Activity "CHILD(REN)"
chip-picker label. This is a **label-only change**: the internal
`children` array, `childIds` field name, and the `deleteChild()` function
(all fixed in place by AD-13's standing rule) are unchanged.

**Rationale:** Tom wants to add non-child family members (e.g. "Dad") the
same way a kid gets added, without the UI implying only children belong
there.

**Alternatives Considered:** A full internal rename (`children` → `members`
everywhere, including the Sheet's Settings tab shape) — explicitly
declined by Tom as out of scope; the risk of touching a name threaded
through GAS, the Claude prompt schema, and cached briefing JSON wasn't
worth it for what is purely a cosmetic change.

**Status:** Designed and prompted (CC_05_FamilyMemberLabelsAndGrouping.md,
2026-08-21) — **not yet deployed.**

---

### AD-23 — Google Calendar Direct Scan Replaces the `.ics` Pipeline for the Briefing

**Decision:** The briefing's `CALENDAR EVENTS` section is no longer fed by the browser-side `.ics` fetch → GAS proxy (`action=ics`) → parse → `&cal=` param pipeline. A new `scanGoogleCalendar(category)` function in `SportsEmailScanner.gs` reads directly from the Google account the script is deployed under (`tjunker9@gmail.com`) via GAS's native `CalendarApp` service, using a two-pass matching strategy:
- **Pass 1** — named sub-calendars (e.g. "Black Lax," "MUHS Lax - Official Offseason...") whose own calendar *name* loosely matches a configured Activity's `name`: every event on that calendar is included automatically, no per-event check needed.
- **Pass 2** — the default/personal calendar, where sports/school events are mixed in with everything else: matched by the event *title* first, then the event *description* as a fallback (catches cases like a ForeTees tee-time confirmation where the activity name only appears in the notes/Reservation ID, never the title).

Events matching neither pass are dropped. Writes into the exact same `CalendarEvents` sheet shape `saveCalendarEvents()` has always used, so `generateSummaryWithClaude()`'s prompt-building logic needed zero changes.

**Rationale:** User's own account already aggregates team/school calendars as named sub-calendars via subscription, making per-team `.ics` URL maintenance in Settings redundant work. Reading `CalendarApp` directly also eliminates CORS/proxy dependency entirely for this data path — a further step past AD-10 (which moved `.ics` fetching server-side but kept `.ics` itself as the source).

**Alternatives Considered:**
- Calendar-name matching only — rejected; several real events (an "Alex - Soccer" personal-calendar entry, a golf tee-time confirmation) don't live on a dedicated named calendar at all, only on the default/personal one.
- Title-only matching on the default calendar — rejected as the sole default-calendar strategy; a real example (a ForeTees confirmation with "foretees-notification-golf" only in the description, not the title) would have been silently dropped.

**Scope:** This replaces the calendar data source for **the briefing only**. The Calendars tab's own week-view UI (`loadAllCals`, `renderCalendar`, `parseICS`, `fetchIcsViaGas`, and the `.ics` URL fields in Settings) is untouched and still `.ics`-based — see ProjectRoadmap backlog for the deferred decision on whether to consolidate that too.

**Status:** Active. Requires a one-time manual Calendar-access authorization grant per deployment (CalendarApp is a new GAS service for this script) — see SessionStarter.md First-Time Setup. Known matching limitation: a calendar named e.g. "WNS Girls Lax Schedule 2…" won't loosely-match an Activity named "WNS Lax" (neither is a substring of the other) — fix is renaming the Activity to more closely match the calendar name, not new matching code. See BestMethods BM-26.

---

### AD-24 — To-Do List Synced to GAS (Closes Out AD-18)

**Decision:** The To-Do list now syncs to a new `Todos` sheet tab via `getTodos()`/`saveTodos()` GAS functions and matching `action=getTodos`/`action=saveTodos` endpoints, mirroring the existing Settings sync pattern (AD-11) exactly — same fire-and-forget push on every save, same last-write-wins trade-off on collision, same "URL is the password" security model (AD-07) with no additional stripping needed since a to-do item carries no secret data.

**Rationale:** AD-18 deliberately deferred this ("revisit if cross-device access becomes a real need") rather than treating it as an oversight. That need arrived — the user reported adding a to-do on desktop and not seeing it on mobile.

**Alternatives Considered:** None beyond what AD-11 already settled for Settings — this is a direct reuse of that pattern, not a new design decision in its own right.

**Status:** Active. **Supersedes AD-18** — the "local-only" decision there is resolved, not just revisited.

---

### AD-25 — Calendar Match Name Split From Activity Display Name

**Decision:** Activities gain a new optional field, `calMatchName`,
separate from the existing `name` field. `scanGoogleCalendar()`'s two-pass
matching (AD-23) compares against `calMatchName` when present, falling
back to `name` when it's empty — exactly today's behavior for any Activity
that doesn't set it.

**Rationale:** This session found that the "Bavarian Soccer" Activity
couldn't match its named Google Calendar because Tom renaming the calendar
in Google Calendar's own UI only changes his personal display of a
calendar he doesn't own — `CalendarApp` still reads the calendar's real
underlying name ("U17/U18 Girls Blue"), which has no substring
relationship to the display name he wants on cards and in the dashboard.
The same root cause already explained the WNS Lax case in ISSUE-020.
Splitting the two purposes into separate fields removes the forced
trade-off between a meaningful display name and a working calendar match.

**Alternatives Considered:**
- Keep forcing Tom to rename Activities to match calendar names exactly —
  works (and is the interim fix in place today) but produces confusing
  display names on cards/briefings for any oddly-named subscribed
  calendar.

**Status:** Designed and prompted (CC_03_CalMatchNameField.md,
2026-08-21) — **not yet deployed.**

---

### AD-26 — Activities Are Archived, Not Deleted

**Decision:** Activities gain a new `archived` boolean field. An archived
Activity is skipped entirely by `scanEmails()`, `scanGroupMe()`, and
`scanGoogleCalendar()` — but its existing history rows are never touched,
filtered, or hidden from any existing read path.

**Rationale:** A hard delete would orphan history rows already tagged with
that Activity's id (per AD-14's design, history intentionally persists
independent of current Settings state). Archiving retires an Activity from
future scanning (e.g. a past season/team) while keeping its history fully
intact and restorable.

**Alternatives Considered:**
- A true delete with cascading history cleanup — rejected as unnecessarily
  destructive and irreversible for what is usually just a seasonal team
  ending, not a mistake to undo.

**Status:** Designed and prompted (CC_04_ArchivedAndFamilyNameFields.md,
CC_07_ArchiveUIAndFamilyName.md, 2026-08-21) — **not yet deployed.**

---

### AD-27 — Editable Per-Deployment Family Name Field

**Decision:** Settings gains a new top-level `familyName` field, synced
the same way as every other Settings field (AD-11). The dashboard's page
title and on-page heading render as `{familyName} Family HQ` when set, a
generic "Family HQ" default otherwise. Subtitle text changes from "Kids
Sports & School Daily Intelligence" to "Family Sports & School Daily
Intelligence."

**Rationale:** First concrete step toward the multi-family, self-hosted
distribution model already decided in AD-19's era of planning
(ProjectRoadmap Phase 6, Model A). Tom also raised personalization/paid
features as a future idea for other families using their own deployment —
not designed or committed to here, just captured (see ProjectRoadmap
backlog for that item).

**Alternatives Considered:** A hardcoded, code-level constant Tom edits
directly per deployment — rejected in favor of a Settings-UI field, since
it's no more effort to build and removes a code-editing step from onboarding
a new family later.

**Status:** Designed and prompted (CC_04_ArchivedAndFamilyNameFields.md,
CC_07_ArchiveUIAndFamilyName.md, 2026-08-21) — **not yet deployed.**

---

### AD-28 — Daily Trigger Now Also Scans Google Calendar Directly

**Decision:** `runDailyBriefing()` (the 6 AM Time-Driven trigger) is
updated to call `scanGoogleCalendar(category)` for each category,
immediately before `scanEmails(category)` — matching the exact call order
`doGet(?action=generate)` already uses (AD-23's Data Flow #2b).

**Rationale:** This session confirmed via a manual `runDailyBriefing` run's
Cloud log that it currently skips the calendar scan entirely — the 6 AM
auto-briefing has been running on whatever calendar data was last written
by a manual Refresh Briefing click, not fresh data. Tom confirmed this
parity gap should be closed.

**Alternatives Considered:** None — this was an unintentional gap between
the two trigger paths, not a deliberate design choice being revisited.

**Status:** Designed and prompted (CC_08_DailyTriggerCalendarScan.md,
2026-08-21) — **not yet deployed.** Flagged for a runtime sanity-check
against GAS's 6-minute execution ceiling once deployed (Gmail scan +
calendar scan + Claude call, times two categories).

---

### AD-29 — Calendars Tab Reads From Scanned Google Calendar Data Instead of Live `.ics` Subscriptions

**Decision:** The Calendars tab no longer live-fetches and client-side-parses `.ics` subscription URLs configured in Settings. A new `action=calendarEvents` backend endpoint (`SportsEmailScanner.gs`) reads the already-persisted `CalendarEvents`/`SchoolCalendarEvents` sheets — the same sheets written by `scanGoogleCalendar()` on every briefing scan (AD-23) — and resolves child labels via the existing `activityLabel()`/`childNames()` join at read time.

**Rationale:** Single source of truth for calendar data; matches the "read from last scan" pattern Email History already uses; removes a redundant data path that had no relationship to the Daily Briefing's own calendar scan and could show a different picture of "what's on the calendar" than Summary did.

**Alternatives Considered:**
- Keep the two pipelines separate — rejected; this is exactly the disconnect that produced ISSUE-020/025-class confusion, and having two independently-sourced calendar views was never a deliberate design choice.

**Status:** Active. The `.ics` subscription pipeline (`fetchIcsServerSide`, `action=ics`) and its Settings UI were removed from the frontend (see PD-12), but the backend handler itself and the `cals` config field were deliberately left in place as unused/dead code rather than deleted, to avoid risk during rollout — see IssuesTracker ISSUE-030 for the future cleanup item.

---

### DD-04 — Typography System Replaced: Quicksand / Nunito Sans (Supersedes DD-01)

**Decision:** Bebas Neue / DM Sans / DM Mono retired project-wide in favor of Quicksand (display/headers) / Nunito Sans (everything else). The uppercase/letter-spaced/monospace treatment previously applied to labels, meta lines, tags, the date badge, and tab labels was dropped in favor of sentence case with little-to-no letter-spacing. Card meta-line separators changed from em dash to middot (e.g. "Alex · Bavarian Soccer · Playmetrics").

**Rationale:** Tom's feedback that the prior system read "too digital." Reviewed via side-by-side interactive mockups of 5 font pairings before selecting Quicksand/Nunito Sans.

**Status:** Active. **Supersedes DD-01** — see DD-01 for the original decision, kept here for history.

---

### PD-09 — Summary Page Filter/Nav Layout Overhaul for Mobile + Desktop Parity

**Decision:** Multiple related UI changes to the Summary tab, reviewed via interactive mockup before implementation:
- The three-tier Category/Child/Activity pill-row filter replaced with a single-row `.filter-row` of dropdowns (Category later split back out into its own control — see PD-10).
- Tab navigation restyled with real active-state styling on desktop, and replaced with a fixed bottom icon nav on mobile (≤640px) since a top tab row isn't thumb-reachable on a phone.
- The Settings tab entry relocated to a header gear icon — no other Settings changes.
- The "Refresh Briefing" button replaced with a small icon-only circular button with a spin-while-loading state.

**Rationale:** Mobile-first usability pass — the prior three-pill-row filter and top-tab nav didn't scale well to a phone screen, especially for one-handed/thumb use.

**Status:** Active.

---

### PD-10 — Category (Sports/School) Control Split Out as Its Own Swipeable/Pill Control

**Decision:** Following PD-09, Category was pulled out of the Family/Activity dropdown row into its own control above it: a swipeable two-segment slider on mobile (tap or swipe to switch), two pill buttons on desktop — both driving the same `setCategory()`/`updateCategoryControlUI()` state.

**Rationale:** Category (Sports vs. School) is a more fundamental, frequently-used switch than Family/Activity filtering — giving it its own dedicated control makes it faster to reach and reduces contention in the combined filter row PD-09 introduced.

**Status:** Active.

---

### PD-11 — Briefing Cards Collapsed by Default, Expandable on Tap

**Decision:** Briefing cards on Summary previously rendered fully expanded (meta, title, tag, full description, bullets always visible), pushing the 7-Day Schedule and Action Items sections far down the page. Cards now render collapsed to a single row (meta/title/tag/chevron) by default, expanding on tap via `toggleCard()` using `scrollHeight`-based animation.

**Rationale:** Gets more of the page's most time-sensitive sections (schedule, actions) above the fold without hiding any card content — just deferring it to a tap.

**Alternatives Considered:**
- Auto-expand urgent-tagged cards by default — rejected; all cards collapse by default regardless of tag/urgency, a deliberate choice for consistency over surfacing urgent items pre-expanded.

**Status:** Active. See IssuesTracker ISSUE-023/024 for two same-session bugs this exposed in the 7-Day Schedule/Action Items sections, which hadn't originally been designed to also default-collapsed.

---

### PD-12 — Calendar Subscriptions Removed From Settings

**Decision:** Per AD-29, the `.ics` subscription config UI (label/URL/linked sport/color per calendar) was removed from Settings entirely, since nothing reads it anymore.

**Rationale:** Dead UI for a data path the Calendars tab no longer uses — leaving it in Settings would let a user configure something with no effect.

**Status:** Active. The backend's `cals` config field and `action=ics` handler were left in place, unused, rather than removed in the same pass — see IssuesTracker ISSUE-030.

---

## Session Notes

> Add entries when decisions are made, revisited, or reversed.

- **Session 1 (Mar 2026):** Log initialized. Decisions AD-01 through AD-09, DD-01 through DD-03, PD-01 through PD-02 documented from MVP state.
- **Session — 2026-08-17:** School HQ delivered as a Unified dashboard (PD-03, superseding PD-01). Settings rebuilt around a Child → Activity linking model (AD-12) with cloud sync (AD-11, superseding AD-06). ICS fetching moved server-side through GAS (AD-10, superseding AD-05), which also fixed ISSUE-002. Added a global Child filter (PD-04), later extended to the Summary tab via structured `childIds` tagging in Claude's output. Two real bugs found and fixed with root-caused, test-verified diagnoses: a `document`-shadowing bug blocking Add/Edit/Delete Child (AD-13), and a stale-tag bug in the rolling email history merge (AD-14). See IssuesTracker.md for the corresponding closed issues.
- **Session — 2026-08-17 (later session):** Unified the previously-separate per-sender/per-calendar-name filters into one Category→Child→Activity cascade (PD-05). Moved scan window to a global setting (AD-15). Fixed the briefing surfacing already-passed content, with both a prompt rule and a deterministic client-side backstop (AD-16). Removed the need for Claude to infer calendar dates by passing real computed ISO dates through instead (AD-17), fixing a real bug (ISSUE-013) this exposed. Added a persistent, local-only To-Do list (AD-18). Root-caused and resolved a "Gmail operation not allowed" failure to a wrong-account deployment — logged as a standing operational rule, not a code fix (AD-19). See IssuesTracker.md ISSUE-011 through ISSUE-016 for the corresponding closed issues.
- **Session — 2026-08-18:** Began GroupMe integration (AD-20) and PWA installability plumbing (AD-22). Mid-session, discovered and fully remediated a leaked Anthropic API key: rotated, migrated to Script Properties along with the new GroupMe token (AD-21), and scrubbed from git history. See IssuesTracker ISSUE-017 for the full incident record.
- **Session — 2026-08-19:** Extended the Activity filter to Summary via `activityId` tagging, mirroring PD-04's `childIds` pattern (PD-06); added a "Child — Sport — Source" briefing card header (PD-07). Fixed Email History's display (Activity name shown per row, found-count reflects active filters) — see IssuesTracker ISSUE-018. Fixed GroupMe rows always showing the literal label "GroupMe" instead of the configured group name, plus a Settings source-count gap that omitted GroupMe groups — see IssuesTracker ISSUE-019. Synced the To-Do list to GAS, closing out AD-18 (AD-24). Replaced the entire `.ics` calendar pipeline feeding the briefing with a direct, server-side Google Calendar scan using hybrid calendar-name + title/description matching (AD-23) — a user-requested architecture change, not a bug fix. Backfilled BestMethods BM-22 through BM-24, which had been referenced by AD-21/ISSUE-017 since 2026-08-18 but never actually written.
- **Session — 2026-08-21:** Confirmed the AD-23 Google Calendar scan
  working end-to-end; root-caused the one real failure (a subscribed
  calendar's display rename doesn't change what `CalendarApp` reads) and
  designed a durable fix (AD-25). Designed archiving in place of deleting
  Activities (AD-26), an editable per-deployment family name (AD-27), and
  closed a parity gap where the 6 AM trigger skipped the calendar scan
  entirely (AD-28). Relabeled "Children" to "Family members" in the UI,
  label-only (PD-08). All four decisions are designed and prompted via 8
  CC prompts generated this session — none deployed yet as of this entry.
- **Session — 2026-08-26:** Full UI/PWA polish pass on Summary and Calendars, all deployed and confirmed working. Summary: filter/nav layout overhaul (PD-09), Category pulled into its own swipeable/pill control (PD-10), briefing cards collapsed-by-default (PD-11). Project-wide typography swap to Quicksand/Nunito Sans, superseding DD-01 (DD-04). Calendars: migrated off the `.ics` subscription pipeline onto scanned Google Calendar data via a new `action=calendarEvents` endpoint (AD-29), with Calendar Subscriptions removed from Settings (PD-12). Six bugs found and fixed along the way — see IssuesTracker ISSUE-023 through ISSUE-028.
