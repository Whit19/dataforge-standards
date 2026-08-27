# Game Day HQ — Project Session Starter
**Protocol:** Load MASTER_CLAUDE_PROTOCOL.md before this file.
Repo: github.com/Whit19/dataforge-standards

## ⚠ Deployment Account — Read This First
Always deploy the Web App from `tjunker9@gmail.com` — that's the account whose Gmail inbox the script actually scans. Deploying from any other Google account silently binds the deployment to that account instead (Apps Script's "Execute as: Me" locks to whoever was logged in at Deploy time), with no error until Refresh Briefing is clicked later. See IssuesTracker ISSUE-016 / DecisionLog AD-19 for the full diagnosis.

## Current Status (as of Aug 27, 2026 session)

Full visual/functional pass across Summary, Calendars, Email History, and
To-Do — all deployed and confirmed working by Tom via live testing.

- **Sports/School accent color system** (PD-13): a `--accent`/
  `--accent-light`/`--accent-mid` CSS variable trio, blue by default,
  swapped to amber via a `body.cat-school` class toggled in
  `updateCategoryControlUI()`. Applied across Summary, Calendars, and
  Email History chrome. To-Do deliberately does NOT follow this system —
  it uses a separate fixed-green `--todo-accent` set instead, via a
  `body.tab-todo` class, since to-dos aren't Sports/School-specific.
- **Email History redesign** (PD-14): dropped the old colored source-app/
  activity pill badges entirely. Rows now show family member name(s) —
  colored via their own `kids[].color`, reusing the `childColorSpansForIds()`
  helper originally built for Calendars — followed by activity name in the
  current accent color, subject on its own line, date top-right. Filtering
  and tap-to-expand-full-body are unchanged.
- **To-Do redesign** (PD-15): optional due dates, grouped into ⚠️ Overdue /
  📅 Upcoming / No due date / Done, click-to-edit in place (text + date,
  no modal), Sports/School filter controls hidden on this tab (never
  applied to to-dos), moved in nav order to sit between Summary and
  Calendars (both desktop tab-nav and mobile bottom-nav). To-Do's
  already-existing cross-device sync (`getTodos`/`saveTodos` → a `Todos`
  sheet) was confirmed working, not rebuilt — the due-date field just
  rides along in the same opaque JSON blob.
- **Daily Briefing refresh hardening** (AD-30): the single ~90s
  `action=generate` call was split into four independently-retryable
  steps (`scanCalendar` → `scanEmail` → `scanGroupMe` → `generateSummary`),
  each with its own elapsed-time log line, driving a real progress bar on
  both the desktop refresh button and mobile pull-to-refresh. Replaces an
  earlier poll-based patch that papered over (rather than fixed) mobile
  connection drops causing false "failed to reach Apps Script" errors
  when the backend had actually succeeded.
- **Manual Updates tab removed entirely** (PD-16) — confirmed dead UI
  (the seven textareas were never read anywhere in the file), closing
  ISSUE-001 by removal rather than by wiring it up. Settings' placement
  was evaluated against moving into the freed nav slot and deliberately
  left in the header gear icon (reaffirms PD-09's mobile-ergonomics
  rationale rather than reversing it).
- **Sender-attribution bug found and fixed in two passes** (AD-31): emails
  from shared-platform senders (e.g. multiple orgs all sending from
  `*.sportsengine.com`) were being attributed to whichever activity's
  sender entry happened to be checked first, regardless of which org
  actually sent the email. Fixed by (1) making `detectApp()` prefer the
  most specific (longest) matching string instead of the first match, then
  (2) widening the search from the From header only to From + Subject
  combined, since some notification templates use a generic display name
  with the real org name only in the subject line.
- Several small bugs caught and fixed along the way: duplicate headers on
  Daily Briefing cards and Calendars events, a static-markup-vs-JS-render
  gap where a filter-dropdown text fix only touched the HTML and not the
  functions that immediately overwrite it, and a To-Do cross-device sync
  gap where "Load Settings from Cloud" never also pulled To-Dos.
- **Doc audit finding, not new work this session:** while updating these
  docs, direct inspection of the live `SportsEmailScanner.gs` confirmed
  AD-25 (`calMatchName`), AD-26 (Activity archiving), AD-27 (`familyName`
  field — visibly live in every screenshot this session as the
  "{Family} Family HQ" title), and AD-28 (daily-trigger calendar-scan
  parity) are all already deployed and active, not "designed, not yet
  deployed" as DecisionLog previously stated. All four were apparently
  shipped together around 2026-08-21 (git commit `cbdc0b6`) but the docs
  were never updated to reflect it — corrected in this pass, see
  DecisionLog for each entry's updated status.

## Next Session's Priorities

1. **Settings UI redesign** — Tom's stated next focus. Full mockup-and-review
   pass, same process as every other page this session and last. Known
   scope so far: add a color-picker to the Activity card (ISSUE-029),
   general layout modernization to match the current typography/design/
   accent system (Settings hasn't been touched since before the Aug 26
   session, so it's visually inconsistent with the rest of the app now).
   Also worth deciding during this pass: now that Calendars no longer uses
   each Activity's `color` field for its event bar (removed Aug 26 session
   — bars follow the accent instead), is that field still worth keeping in
   the Activity card UI at all?
2. Carried over from before, still open/deferred, none touched this
   session: relative-date resolution bug (ISSUE-021), dead `.ics`/
   `action=ics` backend code cleanup (ISSUE-030). (`calMatchName` (AD-25)
   and daily briefing calendar-scan parity (AD-28), both previously listed
   here as open, are confirmed already deployed — see Current Status
   above and DecisionLog.)

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
  ├── On Refresh Briefing (desktop button or mobile pull-to-refresh): four
  │     separately-retryable GET calls in sequence — action=scanCalendar,
  │     action=scanEmail, action=scanGroupMe, action=generateSummary (cat=
  │     {active} on each) — replacing a single combined action=generate
  │     call (AD-30). Each step logs its own elapsed time server-side; the
  │     dashboard shows a real progress bar across all four, and retries
  │     just the failed step (up to 3x) rather than the whole sequence.
  ├── Calendars tab reads scanned Google Calendar data via
  │     GET /exec?action=calendarEvents&cat={active} — no separate "Load
  │     Calendars" step; loads automatically on tab switch, same
  │     CalendarEvents/SchoolCalendarEvents sheets the briefing scan
  │     writes to (AD-23, AD-29). The old .ics-based week view is gone.
  ├── On Email History tab: GET /exec?action=history&cat={active}
  └── Settings sync: GET /exec?action=getSettings / ?action=saveSettings
        (Load Settings from Cloud also pulls To-Dos alongside Settings —
        both fire together, from both the button and the auto-pull-on-load
        path for an unconfigured device)

Google Apps Script (SportsEmailScanner.gs)
  │
  ├── doGet(?action=scanCalendar&cat=)   → scans Google Calendar directly (AD-23), writes CalendarEvents/SchoolCalendarEvents
  ├── doGet(?action=scanEmail&cat=)      → scans Gmail for that category's Activities' senders, writes EmailHistory (AD-30, AD-31)
  ├── doGet(?action=scanGroupMe&cat=)    → scans configured GroupMe groups
  ├── doGet(?action=generateSummary&cat=) → calls Claude on the already-scanned data, stores Summary
  ├── doGet(?action=calendarEvents&cat=) → returns scanned Calendar sheet data for the Calendars tab's week view
  ├── doGet(?action=generate&cat=)  → the original combined version of the 4 steps above, in one execution — kept
  │     server-side for compatibility, no longer called by the dashboard (AD-30)
  ├── doGet(?action=history&cat=)   → returns that category's EmailHistory sheet as JSON
  ├── doGet(?action=ics&url=)       → fetches one .ics feed server-side — retired, no longer called by the dashboard (AD-29), left in place unused
  ├── doGet(?action=getSettings)    → returns Settings sheet JSON (API key always stripped)
  ├── doGet(?action=saveSettings&settings=) → writes Settings sheet (API key always stripped)
  ├── doGet(?action=getTodos) / (?action=saveTodos&todos=) → To-Do cross-device sync (AD-24); each item may carry an optional dueDate (PD-15)
  ├── doGet(?cat=) default          → returns that category's stored Summary sheet as JSON
  ├── doPost()                      → legacy, unused
  └── Daily trigger at 6 AM        → runs runDailyBriefing() for both categories automatically (calls the same scan/summary functions directly, not through the split doGet actions above — unaffected by AD-30)
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
Nav order (both desktop tab-nav and mobile bottom-nav): Summary, To-Do, Calendars, Email History — Settings is the header gear icon, not a nav tab. Manual Updates is gone (removed entirely, PD-16).

| Tab | Purpose |
|-----|---------|
| ⚡ Summary | Home screen — priority, cards, day-grouped 7-Day Schedule, Action Items (each with a "+ To-Do" button that carries the action's own `dateISO` through as the new to-do's due date). Filterable by Category/Child/Activity. Follows the Sports/School accent color (blue/amber, PD-13). |
| ✅ To-Do | Persistent checklist, synced across browsers via GAS (AD-24). Optional due date per item, grouped into Overdue/Upcoming/No due date/Done, click-to-edit in place. Sports/School switch and Family/Activity filters are hidden on this tab — to-dos aren't filtered by either. Uses a fixed green accent, independent of Sports/School (PD-15). |
| 📅 Calendars | Weekly list view, navigate by week, filterable by the same Category/Child/Activity cascade. Reads scanned Google Calendar data (`action=calendarEvents`) — no `.ics` fetching, loads automatically on tab switch. Event bar follows the Sports/School accent; each event's family member name(s) are colored via their own configured color instead of the old per-Activity color field (PD-13/PD-14). |
| 📬 Email History | Rolling email log, filterable by the same Category/Child/Activity cascade. Rows show family member name(s) (own color) · activity name (accent color) · subject · date — no source/app badges anymore (PD-14). Tap a row to expand the full email body. |
| ⚙ Settings | Web App URL, Children, Activities (collapsed accordion, click to expand/edit), global Scan & Briefing Window, Calendars. Entry point is the header gear icon, not a nav tab (PD-09, reaffirmed PD-16). |

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
- **Daily Briefing refresh split into 4 independently-retryable steps**, replacing a single ~60-140s combined call — fixes a real mobile bug where a phone's screen locking mid-request could kill the client connection while the backend kept running and succeeded anyway, showing a false error. (AD-30, closes ISSUE-004)
- **Sender matching now prefers the most specific match, across From + Subject** — `detectApp()` previously returned the first array match on the From header only, which meant activities sharing a broad platform domain (e.g. multiple orgs all on `*.sportsengine.com`) could silently steal each other's emails depending on activity order, and some notification templates with a generic From display name (real org name only in the Subject) couldn't be matched at all. Now resolves to the longest matching string found in From+Subject combined, independent of activity order. (AD-31, closes ISSUE-035)

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
- ~~Rework Manual Updates to be Activity-driven and actually wire it into the prompt~~ — moot as of 2026-08-27; the tab was removed entirely instead (PD-16), not reworked.
- ~~Refresh Briefing timeout/wait-time handling~~ — resolved 2026-08-27 via a different fix than the originally-planned polling pattern: split into 4 independently-retryable steps with a real progress bar (AD-30, closes ISSUE-004).
- Email History not refetching after Refresh Briefing (ISSUE-006) — still open, not incidentally fixed by the filter-cascade rework
- Extend the Calendars tab to show more than 7 days out, filterable by kid/sport/school — likely by reusing the Google-Calendar-scanned data from AD-23 (decision deferred, see ProjectRoadmap backlog)
- PWA manifest + service worker (installable) — drafted, dashboard wiring + icons still pending
- GroupMe API integration (direct message pull, no email delay) — backend drafted, blocked on OAuth app registration
- Push notifications / SMS for same-day urgent events
- Weather forecast injected into game day events
- Carpool coordination view
