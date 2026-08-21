# Game Day HQ — Project Session Starter
**Protocol:** Load MASTER_CLAUDE_PROTOCOL.md before this file.
Repo: github.com/Whit19/dataforge-standards

## ⚠ Deployment Account — Read This First
Always deploy the Web App from `tjunker9@gmail.com` — that's the account whose Gmail inbox the script actually scans. Deploying from any other Google account silently binds the deployment to that account instead (Apps Script's "Execute as: Me" locks to whoever was logged in at Deploy time), with no error until Refresh Briefing is clicked later. See IssuesTracker ISSUE-016 / DecisionLog AD-19 for the full diagnosis.

## Current Status (as of 2026-08-21)
**The Google Calendar scan from AD-23 is now confirmed working end-to-end.**
All 9 steps of the prior session's testing checklist passed, with one real
issue found and resolved along the way: the "Bavarian Soccer" and "WNS Lax"
Activities weren't matching their named Google Calendars because renaming
a calendar in Google Calendar's own UI only changes Tom's personal display
of a calendar he doesn't own — CalendarApp still reads the calendar's real
underlying name. Fixed for now by renaming the Activities to match the
calendars' actual names (e.g. "Bavarian Soccer" → "U17/U18 Girls Blue").
A durable fix — a separate `calMatchName` field so the calendar-matching
term can differ from the display name — is designed and prompted (AD-25)
but **not yet deployed**.

Also this session: found and fixed a stale `scanSportsEmails` Time-Driven
trigger (leftover from before the Sports/School unification, failing daily
with "Script function not found") — deleted directly in the Apps Script
Triggers UI, no code change needed. Found a real bug in relative-date
handling: a briefing card resolved "next Tuesday" against today's date
instead of the source message's own send date, producing a wrong date and
garbled text (ISSUE-021, prompt generated, not yet deployed). Confirmed
`runDailyBriefing()` (the 6 AM trigger) does NOT call `scanGoogleCalendar()`
today, unlike the dashboard's on-demand Refresh Briefing — meaning the
daily auto-briefing has been running on stale calendar data; a fix is
prompted (AD-28) but not yet deployed.

**Major design work, not yet built:** a Settings redesign was scoped and
mocked up interactively with Tom, covering: "Family members" replacing
"Children" as a UI label (PD-08, label-only — internal `children`/
`childIds` naming is unchanged); Sport and School activities grouped
together per person instead of two flat lists; an existing-vs-new-activity
flow with a required Display Name and an optional Calendar Match Name;
inline-editable and deletable email-source/GroupMe chips; archiving an
Activity instead of deleting it (AD-26), so history stays intact; and an
editable per-deployment Family Name field (AD-27) that drives the
dashboard's title/subtitle — first groundwork toward eventual
personalization for other families (see ProjectRoadmap backlog). All of
this is written up as 8 CC prompts (see TimeLog/session notes for the
list) — **none have been applied yet.**

### Next planned focus — start here
1. **Review last session's settings changes.** Before doing anything else,
   confirm which of the 8 CC prompts from 2026-08-21 Tom has actually
   applied and deployed since this was written, and what (if anything)
   broke or needs adjustment. Do not assume any of them are live —
   verify first.
2. If not yet applied: walk through applying the 8 CC prompts in the
   documented order (daily-trigger calendar scan and temp-debug removal
   first, then the data-model fields, then the three frontend Settings
   prompts last, since they depend on the data-model changes).
3. Once deployed, spot-check: relative-date resolution on a new
   ambiguous-date message; the 6 AM trigger's next natural run (or a
   manual `runDailyBriefing` test) actually logging a Google Calendar
   match line for both categories; the new Settings UI (family member
   grouping, archive/restore, editable chips, family name field) against
   real data.
4. Longer-standing open items, still untouched: ISSUE-001 (Manual
   Updates), ISSUE-006 (Email History not refetching after Refresh
   Briefing), GroupMe OAuth setup (Phase 5), PWA dashboard wiring + icons
   (Phase 6).

### Testing checklist — Google Calendar scan: COMPLETE (2026-08-21)
All 9 steps from the prior version of this checklist passed. See Decision
Log AD-23's Status note and IssuesTracker ISSUE-020 for the one real issue
this testing surfaced and how it was resolved. This checklist is kept here
for reference only — no further action needed unless new calendar-matching
problems appear.

## What This Is
A personal dashboard for managing multiple kids' sports teams *and* school activities in one place. It aggregates emails from configured senders, syncs calendars, and uses Claude AI (via Google Apps Script) to generate a daily briefing per category (Sports / School) with a priority, app cards, schedule timeline, and action items — filterable by which kid it's about.

---

## Two Output Files (that's all there is — no separate School files)

### 1. `sports-dashboard.html`
A single-file HTML dashboard hosted on GitHub Pages (see Deployment Workflow below). No backend — all logic is in-browser JS + calls to the GAS Web App. Covers both Sports and School via a category switcher, plus a Settings tab where Children/Activities/senders/calendars are configured (nothing is hardcoded here anymore).

### 2. `SportsEmailScanner.gs`
Google Apps Script bound to a Google Sheet ("Game Day HQ"). Deployed as a Web App (Execute as: Me, Access: Anyone). This is the backend for BOTH categories — it scans Gmail, calls the Anthropic Claude API, fetches calendar `.ics` feeds server-side, stores results, and serves them to the dashboard. (The name is a holdover from before School HQ existed — it's not sports-only anymore.)

---

## Architecture

```
Browser (sports-dashboard.html)
  │
  ├── Settings tab: Children → Activities (sport/school, linked child(ren),
  │     senders) → Calendars (each linked to one Activity)
  ├── Global Child filter pill row + Sports/School switch
  ├── On page load: GET /exec?cat={active} → fetches last stored summary
  ├── On Refresh Briefing: GET /exec?action=generate&cat={active} — GAS scans
  │     Google Calendar directly server-side now (AD-23); the dashboard no
  │     longer sends any calendar data in this request
  ├── On Load Calendars (Calendars tab week view ONLY, decoupled from the
  │     briefing — AD-23): GET /exec?action=ics&url={one feed at a time},
  │     fetched server-side through GAS, not browser-side proxies
  ├── On Email History tab: GET /exec?action=history&cat={active}
  └── Settings sync: GET /exec?action=getSettings / ?action=saveSettings

Google Apps Script (SportsEmailScanner.gs)
  │
  ├── doGet(?action=generate&cat=)  → scans Google Calendar directly (AD-23) +
  │     scans Gmail for that category's Activities' senders + calls Claude +
  │     stores summary
  ├── doGet(?action=history&cat=)   → returns that category's EmailHistory sheet as JSON
  ├── doGet(?action=ics&url=)       → fetches one .ics feed server-side (no CORS issue)
  ├── doGet(?action=getSettings)    → returns Settings sheet JSON (API key always stripped)
  ├── doGet(?action=saveSettings&settings=) → writes Settings sheet (API key always stripped)
  ├── doGet(?cat=) default          → returns that category's stored Summary sheet as JSON
  ├── doPost()                      → legacy, unused
  └── Daily trigger at 6 AM        → runs runDailyBriefing() for both categories automatically
```

---

## Google Sheet Structure
The GAS script creates/uses a Google Sheet named **"Game Day HQ"** with these tabs:

**Sports:**
- **SportsEmails** — latest scan results
- **EmailHistory** — rolling deduped log of sports emails, tagged with ActivityId
- **Summary** — stores `generated_at` and `summary_json` (Claude's output, cards/timeline/actions now carry `childIds`)
- **CalendarEvents** — events pushed from dashboard, tagged with ActivityId

**School:**
- **SchoolEmails**, **SchoolEmailHistory**, **SchoolSummary**, **SchoolCalendarEvents** — same shape as above, school category

**Shared:**
- **Settings** — the full Children/Activities/Calendars/senders JSON, synced from the dashboard. Never contains the Anthropic API key.

---

## GAS Configuration

Secrets (`ANTHROPIC_API_KEY`, `GROUPME_ACCESS_TOKEN`) are read from Apps Script's Script Properties at runtime — never hardcoded as consts. See TechnicalArchitecture.md's "GAS Configuration" section and Decision Log AD-21 for the full pattern and rationale (a hardcoded version of this leaked once — see IssuesTracker ISSUE-017 — don't reintroduce it).

Children, Activities (sport/school, which kid(s), days back/forward), email senders, GroupMe groups, and calendars are all configured from the dashboard's Settings tab — there is no `SPORTS_SENDERS` array or `DAYS_BACK`/`DAYS_FORWARD` constant to hand-edit.

---

## Dashboard Tabs
| Tab | Purpose |
|-----|---------|
| ⚡ Summary | Home screen — priority, cards, day-grouped 7-Day Schedule, Action Items (each with a "+ To-Do" button). Filterable by Category/Child/Activity. |
| 📅 Calendars | Weekly list view, navigate by week, filterable by the same Category/Child/Activity cascade, Load Calendars button |
| 📬 Email History | Rolling email log, filterable by the same Category/Child/Activity cascade (no separate per-sender filter row anymore) |
| ✅ To-Do | Persistent checklist, synced across browsers via GAS (AD-24). Add manually, or via "+ To-Do" on any Action Item. |
| ✏️ Manual Updates | Paste text as fallback input — **currently not wired into the briefing prompt at all; see ISSUE-001** |
| ⚙ Settings | Web App URL, Children, Activities (collapsed accordion, click to expand/edit), global Scan & Briefing Window, Calendars |

---

## Settings Setup Flow (in order)
1. **Children** — add each kid's name and pick a color
2. **Scan & Briefing Window** — one global Days Back to Scan / Days Forward in Briefing, applies to every Activity (AD-15 — no longer set per-Activity)
3. **Sports & School Activities** — one accordion card per team/school (collapsed by default; click to expand and edit); pick category, name it, toggle which child(ren) it covers, add its email senders
4. **Calendars** — one row per `.ics` feed, linked to one Activity via a dropdown

Existing pre-Child-model settings auto-migrate into an `(unsorted)` default Activity per category with no children assigned, so nothing breaks — go re-sort into real children afterward.

---

## Key Technical Decisions & Fixes (full detail in DecisionLog.md / IssuesTracker.md)

- **Claude API lives in GAS, not the browser** — CORS. Model: `claude-sonnet-4-5`, `max_tokens: 4000`.
- **Calendar events sent to GAS via GET param** (not POST) — CORS again.
- **ICS calendar fetching moved server-side through GAS** — the old browser-side 3-proxy waterfall (allorigins/corsproxy.io/codetabs) became unreliable in production; `UrlFetchApp` has no CORS restriction and doesn't depend on third-party proxy uptime. (AD-10)
- **Settings sync to a GAS-backed Sheet**, in addition to localStorage, so a new browser only needs the Web App URL to pull everything back down. The Anthropic key is explicitly stripped before anything gets written there. (AD-11)
- **Child → Activity linking model** — Children link to Activities (not directly to senders/calendars), because a team or school is the real unit siblings actually share. (AD-12)
- **Global Child filter**, extended to the Summary tab by having Claude tag its own output with `childIds` from a legend of known children. (PD-04)
- **ICS timezone fix** — events with `Z` suffix (UTC) were parsed as local time → 4-6hr offset; fixed by detecting `Z` and parsing as true UTC.
- **Briefing uses the full rolling email history**, not just the latest scan, so older-but-still-relevant context isn't missed — this is *why* AD-16 (below) was needed, not a contradiction of it.
- **Two bugs found and fixed in the Child→Activity session** (both root-caused with a direct repro/read of the code before fixing, not guessed):
  - A DOM-shadowing bug where inline `onclick`/`oninput` handlers referencing bare `children`/`removeChild` silently resolved to `document.children`/`document.removeChild` instead of the app's own state — broke Add/Edit/Delete Child entirely. (AD-13)
  - The rolling email-history merge kept an email's *original* ActivityId tag forever instead of letting fresh scans correct it, so per-child email filtering only worked for kids whose emails happened to be scanned after their Settings reorg. (AD-14)
- **Category → Child → Activity filter cascade** replaces the old separate per-sender/per-calendar-name filters. (PD-05)
- **Global scan/briefing window**, migrated automatically from old per-activity values. (AD-15)
- **Briefing staleness prevention** — explicit `todayISO` + prompt rule, plus a `dateISO` field and client-side backstop filter. (AD-16)
- **Calendar events carry a real computed ISO date**, so Claude copies it instead of inferring one — fixed a real bug (ISSUE-013) this had been causing. (AD-17)
- **Model-output reliability pattern**, established and reused multiple times (ISSUE-010, ISSUE-011, ISSUE-013, and now ISSUE-020's `activityId`): whenever a prompt asks Claude to compute or tag something, pair it with a deterministic client-side check that doesn't fully trust the model's compliance. (BestMethods BM-20, BM-25)
- **Google Calendar scanned directly, replacing `.ics` for the briefing** — two-pass matching (named-calendar match, then title/description match on the default calendar) via GAS's native `CalendarApp`, no CORS/proxy dependency at all for this data path. (AD-23, BM-26)
- **To-Do list synced to GAS**, closing out the earlier local-only trade-off, using the same pattern as Settings sync. (AD-24)

---

## Deployment Workflow (after any GAS change)
1. **Confirm you're logged into `tjunker9@gmail.com`** before doing anything else — see the warning at the top of this file
2. Paste updated script into Extensions → Apps Script
3. Save (Ctrl+S)
4. Deploy → Manage Deployments → ✏️ edit → New version → Deploy
5. **URL does not change** — no dashboard update needed
6. No need to re-run `installDailyTrigger` unless testing

## Deployment Workflow (after any dashboard HTML change)
1. Commit + push `sports-dashboard.html` (or whatever filename your GitHub Pages repo uses) to the repo
2. GitHub Pages picks it up automatically — no separate build/deploy step

## First-Time Setup (for a new environment)
1. Get an Anthropic API key at console.anthropic.com → add billing credits
2. Open the Google Sheet **while logged into `tjunker9@gmail.com`** → Extensions → Apps Script → paste `SportsEmailScanner.gs`
3. Add the API key at the top of the script — nowhere else
4. Run `installDailyTrigger` once (sets 6 AM daily schedule)
5. **Before deploying:** in the Apps Script editor, select any function in the toolbar dropdown (e.g. `scanGoogleCalendar`) and click Run once manually — approve the Calendar access prompt when it appears. Skipping this means `generate` will silently save zero calendar events (logged, not thrown) rather than fail loudly.
6. Deploy as Web App → Execute as: Me, Access: Anyone — **confirm the account in the top-right of the Apps Script editor is `tjunker9@gmail.com` before clicking Deploy**
7. Open the dashboard → Settings tab → paste the Web App URL
8. Settings → add Children, then Activities (with senders) for Sports and/or School
9. (Optional, only for the Calendars tab's own week view — no longer required for the briefing) link Calendars, then Calendars tab → Load Calendars
10. Summary tab → Refresh Briefing

---

## Possible Next Features
- Rework Manual Updates to be Activity-driven and actually wire it into the prompt (ISSUE-001)
- Refresh Briefing timeout/wait-time handling — polling pattern (ISSUE-004), deliberately deferred, revisit when ready
- Email History not refetching after Refresh Briefing (ISSUE-006) — still open, not incidentally fixed by the filter-cascade rework
- Extend the Calendars tab to show more than 7 days out, filterable by kid/sport/school — likely by reusing the Google-Calendar-scanned data from AD-23 (decision deferred, see ProjectRoadmap backlog)
- PWA manifest + service worker (installable) — drafted, dashboard wiring + icons still pending
- GroupMe API integration (direct message pull, no email delay) — backend drafted, blocked on OAuth app registration
- Push notifications / SMS for same-day urgent events
- Weather forecast injected into game day events
- Carpool coordination view
