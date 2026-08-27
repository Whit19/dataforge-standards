# HQ Dashboard — Technical Architecture

**Last Updated:** August 26, 2026  
**Status:** Unified Sports + School dashboard live, with a Child → Activity linking model and a Category → Child → Activity filter cascade that also filters Summary. The briefing's calendar data comes directly from a server-side Google Calendar scan on-demand (AD-23); the Calendars tab now reads that same scanned data too, via a new `action=calendarEvents` endpoint (AD-29), rather than a live `.ics` fetch. A matching scan on the 6 AM daily trigger, a calendar/display name split (`calMatchName`), Activity archiving, and a per-deployment family name field are designed and prompted but **not yet deployed** — see the "Pending Changes" note below and DecisionLog AD-25 through AD-28.

---

## System Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│  Browser (sports-dashboard.html) — single file, no build step        │
│                                                                        │
│  ┌───────────┐ ┌───────────┐ ┌────────────┐ ┌────────┐ ┌───────────┐ │
│  │ Summary   │ │ Calendars │ │ Email      │ │ Manual │ │ Settings  │ │
│  │           │ │           │ │ History    │ │Updates │ │           │ │
│  │ Fetches   │ │ Reads last│ │ GET ?action│ │ (dead  │ │ Children →│ │
│  │ stored    │ │ Google Cal│ │ =history   │ │ input, │ │ Activities│ │
│  │ JSON, or  │ │ scan via  │ │ &cat=      │ │ see    │ │ → senders │ │
│  │ ?action=  │ │ ?action=  │ │            │ │ISSUE-  │ │ / Calendars│ │
│  │ generate  │ │ calendar- │ │            │ │  001)  │ │           │ │
│  │ (no cal   │ │ Events    │ │            │ │        │ │           │ │
│  │ data sent │ │ (AD-29,   │ │            │ │        │ │           │ │
│  │ — AD-23)  │ │ same as   │ │            │ │        │ │           │ │
│  │           │ │ last-scan │ │            │ │        │ │           │ │
│  │           │ │ pattern)  │ │            │ │        │ │           │ │
│  └───────────┘ └───────────┘ └────────────┘ └────────┘ └───────────┘ │
│                                                                        │
│  Filter cascade: Category → Child → Activity, three pill rows above  │
│  the tabs (PD-05). Now drives Calendars + Email History + Summary    │
│  identically via activityId (PD-06, extending PD-04's childIds-only  │
│  Summary support)                                                     │
│                                                                        │
│  Settings: GAS Web App URL in localStorage only.                     │
│  Children/Activities/Calendars/senders in localStorage AND synced    │
│  to the Sheet's "Settings" tab (AD-11).                              │
└──────────────────────────────┬─────────────────────────────────────┘
                                │ fetch() — allorigins.win fallback only
                                │ if the direct fetch to /exec fails
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Google Apps Script (SportsEmailScanner.gs)                          │
│  Deployed as Web App (Execute as: Me, Access: Anyone)                │
│                                                                        │
│  doGet(?action=generate&cat=sports|school)                            │
│    1. scanGoogleCalendar(category) → reads CalendarApp directly (two-│
│       pass: named-calendar match, then title/description match on   │
│       the default calendar) → writes the category's CalendarEvents   │
│       sheet, same shape as before (AD-23 — replaces the old ?cal=    │
│       param entirely for this purpose)                                │
│    2. scanEmails(category) → Gmail search across ALL that category's │
│       Activities' senders → per-category Emails sheet, tagged with   │
│       each row's ActivityId                                          │
│    3. updateHistorySheet(category) → rolling deduped log, keyed by   │
│       date|subject, where the CURRENT scan's tag always wins on a    │
│       collision (AD-14 — this used to be a bug)                      │
│    4. generateSummaryWithClaude(category) → merges history + cal     │
│       events (each calendar event includes a real {dateISO}, AD-17), │
│       resolves "[Child — Activity]" labels, adds "KNOWN CHILDREN"    │
│       and "KNOWN ACTIVITIES" (id = name) legends + todayISO + a      │
│       no-stale-content rule (AD-16), asks Claude to tag each card/   │
│       timeline/action with childIds, activityId (PD-06), and dateISO │
│       → POST to Claude → parse JSON → storeSummary()                 │
│                                                                        │
│  doGet(?action=history&cat=)      → per-category EmailHistory sheet  │
│  doGet(?action=ics&url=)          → fetch one .ics feed server-side  │
│                                      (AD-10 — replaces the old        │
│                                      browser-side proxy waterfall)    │
│  doGet(?action=getSettings)       → return the Settings sheet's JSON │
│  doGet(?action=saveSettings&…)    → write settings, stripping any    │
│                                      Anthropic API key first (AD-11) │
│  doGet(?cat=) default             → return that category's stored    │
│                                      Summary sheet                    │
│  doPost()                         → legacy, still present, unused    │
│                                                                        │
│  Daily trigger: runDailyBriefing() at 6 AM, runs both categories     │
└──────────────────────────────┬─────────────────────────────────────┘
                                │
          ┌─────────────────────┼───────────────────────────┐
          ▼                     ▼                            ▼
 ┌─────────────────┐  ┌──────────────────┐         ┌──────────────────────┐
 │  Gmail API      │  │ Anthropic API     │         │ Google Sheets         │
 │  (via GAS       │  │ claude-sonnet-4-5 │         │ "Game Day HQ"         │
 │  GmailApp)      │  │ max_tokens: 4000  │         │                       │
 │                 │  │ Returns JSON      │         │ Per category (Sports/ │
 │  Query: ALL     │  │ briefing object,  │         │ School), 4 tabs each: │
 │  senders across │  │ now with childIds │         │ - {Cat}Emails         │
 │  the category's │  │ on cards/timeline/│         │ - {Cat}EmailHistory   │
 │  Activities     │  │ actions           │         │ - {Cat}Summary        │
 │  + date range   │  │                   │         │ - {Cat}CalendarEvents │
 └─────────────────┘  └──────────────────┘         │ plus a shared:        │
                                                     │ - Settings            │
                                                     └───────────────────────┘
```

---

## File Inventory

| File | Type | Role | Status |
|------|------|------|--------|
| `sports-dashboard.html` | Single-file HTML/CSS/JS | Unified Sports + School frontend, incl. Settings UI | ✅ Live |
| `SportsEmailScanner.gs` | Google Apps Script | Unified Sports + School backend / AI orchestration | ✅ Live |
| `manifest.json` | Static JSON | PWA web manifest — installability on desktop + mobile from one URL | 🔧 Drafted, uncommitted |
| `sw.js` | Service Worker | PWA installability + safe offline fallback; deliberately network-first for the dashboard itself (AD-22) | 🔧 Drafted, uncommitted |

There is no separate `school-dashboard.html` / `SchoolEmailScanner.gs` — see Decision Log PD-03 for why that plan was reversed.

---

## GAS Configuration — Script Properties, Not Hardcoded Consts (AD-21)

Both `ANTHROPIC_API_KEY` and `GROUPME_ACCESS_TOKEN` are read at runtime from Apps Script's Script Properties, never assigned as literal strings:

```javascript
const ANTHROPIC_API_KEY = PropertiesService.getScriptProperties().getProperty('ANTHROPIC_API_KEY');
const GROUPME_ACCESS_TOKEN = PropertiesService.getScriptProperties().getProperty('GROUPME_ACCESS_TOKEN');
```

Both values are set manually via the Apps Script editor's **Project Settings (gear icon) → Script Properties** — never through code, never through the dashboard's Settings sync. This is a hard rule, not a preference: an earlier version of this script had `ANTHROPIC_API_KEY` as a hardcoded const, which was committed to git and leaked (see IssuesTracker ISSUE-017). Script Properties keeps the secret out of any file that could ever be committed, regardless of the repo's visibility.

Children, Activities (sport/school, which kid(s), days back/forward), email senders, GroupMe groups, and calendars are all configured from the dashboard's Settings tab — there is no `SPORTS_SENDERS` array or `DAYS_BACK`/`DAYS_FORWARD` constant to hand-edit anymore.

---

## Settings Data Model (Decision Log AD-12, AD-15)

```
children: [
  { id, name, color }
]

activities: [
  {
    id, category: 'sports'|'school', name,
    calMatchName,                  // PENDING (AD-25) — optional; used
                                    // only for Google Calendar matching,
                                    // falls back to `name` when empty.
                                    // Not yet deployed.
    archived,                      // PENDING (AD-26) — optional
                                    // boolean, default false. Skips
                                    // this Activity in all scanning
                                    // when true; history is never
                                    // filtered. Not yet deployed.
    childIds: [id, ...],           // 0, 1, or many — siblings can share a team/school
    color,
    senders: [ { match, app, color } ]
    groupmeGroups: [ { groupId, name, color } ],  // NEW (AD-20) — parallel to senders, not merged into it
    // daysBack/daysForward REMOVED from here (AD-15) — old saved data may
    // still have them; they're simply ignored, not actively stripped
  }
]

cals: [
  { name, url, color, activityId }  // '' = unlinked
]

maxEmails: 100
daysBack: 15     // NEW (AD-15) — global, applies to every Activity
daysForward: 7   // NEW (AD-15) — global, applies to every Activity
familyName: ''   // PENDING (AD-27) — optional; drives the dashboard's
                 // page title and heading when set. Not yet deployed.
```

- A calendar or Activity is linked to child(ren) via `activityId` → `Activity.childIds`, never a direct child tag on the raw sender/calendar row.
- Both the Gmail scan (`scanEmails`) and the Sheet-side `EmailHistory`/`CalendarEvents` rows carry `ActivityId` as their last column.
- `migrateLegacySettings()` (GAS) / `migrateLegacyLocal()` (HTML) detect the pre-AD-12 flat shape (`{sports:{...}, school:{...}, cals:[{category}]}`) and wrap it into an `(unsorted)` default Activity per category with `childIds: []`, so upgrading never loses existing configuration.
- **Never add `ANTHROPIC_API_KEY` to this object.** `saveSettings()` explicitly deletes it if present before writing to the Sheet (see AD-11) — that Sheet is world-readable to anyone with the Web App URL.

---

## Data Flows

### 1. Daily Auto-Briefing (6 AM)
```
GAS trigger → runDailyBriefing()
  → for category in ['sports', 'school']:
      scanGoogleCalendar(category) — AD-28, see note below
      scanEmails(category) → Gmail → {Cat}Emails sheet + {Cat}EmailHistory sheet
      scanGroupMe(category) → {Cat}EmailHistory sheet (AD-20)
      generateSummaryWithClaude(category)
        → read {Cat}EmailHistory sheet (merged with this scan, fresh data wins — AD-14)
        → read {Cat}CalendarEvents sheet (just written by scanGoogleCalendar() above)
        → build category-specific prompt, with a KNOWN CHILDREN legend
        → POST to Claude API
        → storeSummary(category) → {Cat}Summary sheet
```

**AD-28 confirmed deployed (2026-08-27 doc audit):** this section previously
described `runDailyBriefing()` as NOT calling `scanGoogleCalendar()` — a gap
designed and prompted 2026-08-21 but never confirmed live. Direct inspection
this session found `scanGoogleCalendar(category)` already called first, matching
`doGet(?action=generate)`'s call order — apparently shipped with the same
2026-08-21 batch (git `cbdc0b6`) but never marked as such in either this file
or DecisionLog. Corrected in both places this session. This flow calls the
underlying functions directly, independent of the `action=generate` split
described in Data Flow #2 — it isn't affected by that change.

### 2. On-Demand Refresh from Dashboard (AD-30, 2026-08-27 — supersedes single `action=generate` call)
```
User clicks Refresh Briefing (desktop button or mobile pull-to-refresh) → generate() in JS
  → runGenerateSteps(category, signal) drives 4 separate requests in sequence,
    each independently retryable (fetchStepWithRetry(), up to 3 attempts, 2s
    apart — every step is idempotent server-side, so a dropped connection
    just retries that same step):
      1. GET /exec?action=scanCalendar&cat={active}   → scanGoogleCalendar(category)
      2. GET /exec?action=scanEmail&cat={active}       → scanEmails(category)
      3. GET /exec?action=scanGroupMe&cat={active}     → scanGroupMe(category)
      4. GET /exec?action=generateSummary&cat={active} → generateSummaryWithClaude(
           category, [] ) — passes [] for emailRows deliberately; the function
           already merges the full {Cat}EmailHistory sheet itself (the same
           sheet step 2 just wrote), so there's no need to thread rows across
           what are now separate stateless requests
  → each step call shows real progress client-side (label + fill bar, e.g.
    "Scanning email… (2/4)", shared between the desktop button and mobile
    pull-to-refresh)
  → step 4's response — { summary, generatedAt }, same shape action=generate
    always returned — → renderOutput() → cached as lastBriefingData
    → renderBriefingFiltered() applies the active Child AND Activity filters
      (PD-04, PD-06) AND a deterministic isPastISO() backstop client-side (drops
      any past-dated item regardless of what Claude returned, AD-16)
      → renderTimelineGrouped() buckets the 7-Day Schedule by day using each
        item's dateISO
```
**Retired path, kept for compatibility:** the original combined `GET /exec?
action=generate&cat=` handler (running all 4 steps in one GAS execution) is
still present server-side, but no longer called by the dashboard — see API
Contracts below and DecisionLog AD-30 for why (a single ~60-140s connection
was too exposed to a phone's screen locking mid-request, ISSUE-004).

### 2b. Google Calendar Scanning (AD-23)
```
GAS: scanGoogleCalendar(category) — called from doGet(?action=generate),
immediately before scanEmails(category)
  → Pass 1: CalendarApp.getAllCalendars(), excluding the default calendar
      → for each calendar whose NAME loosely matches (bidirectional substring)
        a configured Activity's `name`: include every event on that calendar
        in the window, tagged with that Activity's id — no per-event check
  → Pass 2: CalendarApp.getDefaultCalendar().getEvents(start, end)
      → for each event: match by TITLE against configured Activity names first,
        then by DESCRIPTION as a fallback (catches e.g. a ForeTees confirmation
        where the activity name is only in the notes)
      → no match on either → event is dropped, not included
  → events sorted by dateISO, capped at 60 → saveCalendarEvents(category, events)
    — the SAME function and sheet shape the old ?cal= param path always used, so
    generateSummaryWithClaude()'s calSection prompt-building needed no changes
```

### 3. Calendar Loading (reads from the last Google Calendar scan — same load-once-per-session, cache-client-side pattern as Email History's Data Flow #4, AD-29)
```
User opens Calendars tab
  → loadCalendarEvents() → fetch GET /exec?action=calendarEvents&cat={active}
  → GAS: calendarEvents(cat) → reads {Cat}CalendarEvents sheet (already written
    by scanGoogleCalendar() on the last briefing scan, AD-23/2b — no live fetch
    here, same "read from last scan" pattern as loadHistory())
    → resolves childLabel per event via the same activityLabel()/childNames()
      join generateSummaryWithClaude() already performs
    → detects Date-typed cells (Sheets auto-coerces date/time-shaped strings
      to real Date objects on write — see BestMethods) and reformats them to
      plain yyyy-MM-dd / h:mm a strings before returning (ISSUE-025)
  → returns { summary, team, childLabel, activityId, date, time, location,
    dateISO } per event
  → renderActivitySwitch() → renderCalendar(), which applies category +
    child + activity filters together (categoryFilteredEvents() → filteredEvents(),
    PD-05 — replaces the old per-calendar-name team filter)
```

**Retired (superseded by AD-29, 2026-08-26):** Calendars previously ran a fully separate pipeline — live-fetching and client-side-parsing `.ics` subscription URLs configured in Settings via `normalizeIcsUrl()` → `fetchIcsViaGas()` (`GET /exec?action=ics&url=...`, server-side fetch per AD-10) → `parseICS()` (split by VEVENT, parse DTSTART/SUMMARY/LOCATION). This was disconnected from the Daily Briefing's own `scanGoogleCalendar()` scan and could show a different picture of "what's on the calendar" than Summary did. The frontend calls and the Calendar Subscriptions Settings UI were removed (PD-12); the backend `fetchIcsServerSide()`/`action=ics` handler and the `cals` config field were deliberately left in place, unused, rather than deleted — see IssuesTracker ISSUE-030 for the future cleanup item.

### 3b. GroupMe Message Scanning (AD-20)
```
GAS: scanGroupMe(category) — called from both runDailyBriefing() and
doGet(?action=generate), immediately after scanEmails(category)
  → for each Activity in category, for each configured groupmeGroups entry:
      GET api.groupme.com/v3/groups/{groupId}/messages?token=GROUPME_ACCESS_TOKEN
      → filter messages by the same global daysBack window as email
      → normalize each message into the SAME row shape as email:
        [msgDate, senderName, 'GroupMe', textPreview, fullText, activityId]
  → updateHistorySheet(category, groupmeRows) — the SAME function email uses,
    called a second time in the same run; since it always re-reads the sheet
    fresh and merges newRows on top, this produces the same result as one
    combined call
  → generateSummaryWithClaude() and both prompt builders need NO changes —
    GroupMe messages are indistinguishable from email rows by the time they
    reach the prompt, other than App: 'GroupMe'
```

### 4. Email History Browsing
```
User opens Email History tab
  → loadHistory() → fetch GET /exec?action=history&cat={active}
  → GAS reads {Cat}EmailHistory sheet → returns JSON array (each row incl. activityId)
  → renderHistory() → filters by the active Child (resolving activityId → Activity →
    childIds) AND the active Activity (direct activityId match) — PD-05, replaces the
    old per-sender/per-app filter row entirely (ISSUE-006 note: this changed WHAT
    Email History filters by, not WHEN it refetches — that gap is still open)
```

### 5. Settings Sync
```
Local edit (add/edit/remove Child, Activity, sender, or Calendar)
  → saveSettingsLocal() → localStorage['gdhq_settings']
  → (only on explicit Save) pushSettingsToGas() → GET /exec?action=saveSettings&settings=[JSON]
      → GAS strips any API key field, writes to the Settings sheet

Fresh browser, GAS URL already known, no local settings
  → auto-triggers pullSettingsFromGas(silent) AND pullTodosFromGas(silent) once
    on load (ISSUE-034, 2026-08-27 — previously only settings were pulled here,
    leaving a fresh device's To-Do list empty until something else happened to
    trigger a todos pull)

Manual "☁ Load Settings from Cloud" button
  → pullSettingsFromGas(false) AND pullTodosFromGas(false) — same ISSUE-034 fix,
    applied to the manual path too
```

### 6. To-Do List (AD-18, synced to GAS as of AD-24; due dates + grouping + inline editing as of PD-15)
```
Add/toggle/delete a To-Do item
  → loadTodos()/saveTodos() → localStorage['gdhq_todos'] AND, as of AD-24,
    pushTodosToGas() → GET /exec?action=saveTodos&todos=[JSON], fire-and-forget,
    same last-write-wins pattern as Settings sync (AD-11). Array of
    { id, text, dueDate, done, addedAt } — dueDate (PD-15, optional, "" or
    absent for older items) added with no backend schema change needed, since
    getTodos/saveTodos store the array as opaque JSON.
  → renderTodos() groups into Overdue (dueDate set + isPastISO) / Upcoming
    (dueDate set + future) / No due date / Done, each sorted by dueDate, and
    re-renders the To-Do tab's DOM
  → startEditTodo()/saveEditTodo() (PD-15): clicking a row swaps it for inline
    text + date inputs and a Save button — no modal, edits both fields in place

On page load
  → pullTodosFromGas(silent) → GET /exec?action=getTodos → overwrites local
    todos with the cloud copy (via saveTodosLocalOnly(), which does NOT push
    back up) → renderTodos()

Action Item "+ To-Do" button (Summary tab)
  → addTodoFromAction(idx) → dedupes by exact text match against existing todos,
    carries the action's own dateISO through as the new to-do's dueDate (PD-15
    — no free-text date parsing), then calls saveTodos() (which now also syncs
    to GAS) + renderTodos() (a same-session bug, ISSUE-014, was fixed here —
    renderTodos() was originally missing after saveTodos())
```

---

## API Contracts

### GAS Web App Endpoints

**GET /exec?cat=sports|school** — Returns that category's stored briefing
```json
{
  "summary": {
    "priority": "string",
    "cards": [{ "source": "", "sourceColor": "#hex", "title": "", "body": "", "bullets": [], "tag": "urgent|today|soon|info", "childIds": ["..."] }],
    "timeline": [{ "date": "Mon Mar 10", "dateISO": "2026-03-10", "time": "3:30 PM", "event": "", "childIds": ["..."] }],
    "actions": [{ "text": "string", "dateISO": "2026-03-10 or empty string", "childIds": ["..."] }]
  },
  "generatedAt": "ISO 8601 string"
}
```
Note: `actions` were plain strings before the child-tagging change; the dashboard's `renderBriefingFiltered()` accepts both shapes so old cached briefings still render. `dateISO` (AD-16) is likewise absent on any briefing cached before this session's change — the dashboard's `isPastISO()`/`renderTimelineGrouped()` treat a missing `dateISO` as "leave alone" (shown under an "Other" bucket, never dropped), not as an error.

**GET /exec?action=generate&cat=** — Regenerates that category's briefing in one GAS execution (scan Calendar → scan Email → scan GroupMe → call Claude, all sequential), returns same shape as the default endpoint above. **Retired from frontend use (AD-30, 2026-08-27)** — no longer called by the dashboard, kept server-side for compatibility. Superseded by the 4 endpoints below, split out so the client gets real per-step progress and each connection is short enough to survive a phone backgrounding mid-request (ISSUE-004).

**GET /exec?action=scanCalendar&cat=** → `scanGoogleCalendar(category)`, returns `{ ok: true }`
**GET /exec?action=scanEmail&cat=** → `scanEmails(category)`, returns `{ ok: true }`. Sender attribution (`detectApp()`, called once per message) is longest-match-wins across `From + ' ' + Subject` combined as of AD-31 — previously first-array-match across `From` only, which let a broad shared-platform domain entry (e.g. `sportsengine.com`) silently claim emails that actually belonged to a different Activity on the same platform, either because a more specific entry lost on array order or because the identifying org name only appeared in the Subject line, never in From.
**GET /exec?action=scanGroupMe&cat=** → `scanGroupMe(category)`, returns `{ ok: true }`
**GET /exec?action=generateSummary&cat=** → `generateSummaryWithClaude(category, [])`, returns `{ summary, generatedAt }` (falls through to the same default-endpoint response code rather than duplicating it)

All 4 (AD-30) run the exact same 4 functions `action=generate` always called, in the same order, just as separately-addressable requests — each is idempotent (safe to retry; re-scanning/re-generating just overwrites the same sheet). The dashboard drives them in sequence via `runGenerateSteps()`, with per-step retry (`fetchStepWithRetry()`, up to 3 attempts) on the client. `generateSummary` passes `[]` for emailRows deliberately — see Data Flow #2 above.

**GET /exec?action=history&cat=[&app=AppName]** — Returns email history
```json
{ "emails": [{ "date": "ISO", "from": "", "app": "", "subject": "", "body": "", "activityId": "" }] }
```

**GET /exec?action=ics&url=...** — Fetches one `.ics` feed server-side, category-agnostic. **Retired (AD-29, 2026-08-26)** — no longer called by the frontend; left in place unused, see IssuesTracker ISSUE-030.
```json
{ "ok": true, "text": "raw ICS content" }
```

**GET /exec?action=calendarEvents&cat=sports|school** — Returns that category's scanned Google Calendar events (AD-29), reading the same `CalendarEvents`/`SchoolCalendarEvents` sheets `scanGoogleCalendar()` writes on every briefing scan
```json
{ "events": [{ "summary": "", "team": "", "childLabel": "", "activityId": "", "date": "Thu, Aug 20", "time": "3:30 PM", "location": "", "dateISO": "2026-08-20" }] }
```
`childLabel` is resolved server-side via the same `activityLabel()`/`childNames()` join `generateSummaryWithClaude()` already uses. Date/time-shaped cell values are explicitly reformatted from any Sheets-coerced `Date` object back to plain strings before returning — see ISSUE-025.

**GET /exec?action=getSettings** — Returns the Settings sheet's JSON (never includes the Anthropic key)

**GET /exec?action=saveSettings&settings=JSON** — Writes settings (strips any Anthropic key field first)

**POST /exec** — Legacy calendar-events endpoint, retained but unused by the current dashboard

### Calendar Events Sheet Format (written server-side by scanGoogleCalendar(), previously sent as a GET param from the dashboard — AD-23)
```json
{ "summary": "title", "team": "Activity name", "date": "Thu, Aug 20", "time": "3:30 PM or All Day", "location": "location", "activityId": "...", "dateISO": "2026-08-20" }
```
As of AD-23, this is built entirely server-side by `scanGoogleCalendar()` and written directly via `saveCalendarEvents()` — it is no longer sent as a `?cal=` GET param from the dashboard (the old 60-event/4000-char URL-length cap, BM-10, no longer applies here since there's no URL involved; `scanGoogleCalendar()` still caps at 60 for prompt-size reasons, not URL-length ones). `dateISO` is computed directly from each `CalendarApp` event's real start time via `Utilities.formatDate()` — same "pass the real value through, don't make Claude compute it" principle as AD-17 originally established for the old `.ics`-sourced pipeline.

### Claude Prompt Schema
Claude is prompted (per category, via `buildSportsPrompt`/`buildSchoolPrompt`) to return ONLY valid JSON:
```json
{
  "priority": "string",
  "cards": [{ "childIds": ["..."], "activityId": "...", "...": "..." }],
  "timeline": [{ "childIds": ["..."], "activityId": "...", "dateISO": "YYYY-MM-DD", "...": "..." }],
  "actions": [{ "text": "...", "dateISO": "YYYY-MM-DD or empty string", "childIds": ["..."], "activityId": "..." }]
}
```
Tag values (sports): `urgent` | `today` | `soon` | `info`. Tag values (school): `urgent` | `today` | `soon` | `exam` | `info`.  
`childIds` must use only the exact ids from the `KNOWN CHILDREN (id = name)` legend given in the prompt; an empty array means general/unclear, and the dashboard treats that as "show under every filter." `activityId` (PD-06) follows the identical rule against a `KNOWN ACTIVITIES (id = name)` legend — an empty string means general/unclear, same graceful-degradation treatment. `dateISO` (AD-16) must never describe a date already fully passed relative to the prompt's `todayISO` — enforced both by an explicit prompt rule and, since model instruction-following isn't guaranteed, a deterministic client-side `isPastISO()` filter that drops any past-dated item regardless of what Claude returned. For calendar-sourced timeline items, Claude is instructed to copy `dateISO` directly from the CALENDAR EVENTS section rather than compute it — that value now comes from `scanGoogleCalendar()` (AD-23) rather than the old `.ics`-sourced `?cal=` payload (AD-17), but the "pass the real value through" principle is unchanged.

---

## Key Technical Constraints

| Constraint | Impact | Solution |
|-----------|--------|----------|
| CORS blocks browser → Claude API | Can't call Claude from browser | All AI calls go through GAS (AD-01) |
| CORS blocks browser → GAS (sometimes) | Fetch failures | allorigins.win proxy fallback, `/exec` only |
| GAS GET URL length limit ~8kb | Applied to calendar data sent from the browser before AD-23; calendar data is now built entirely server-side (`scanGoogleCalendar()`) and never sent as a GET param, so this constraint no longer applies to calendar events specifically — still relevant for any other data that might be sent this way | Historical: capped events at 60, short keys, 4000 char limit. `scanGoogleCalendar()` still caps at 60, now for Claude prompt-size reasons, not URL-length |
| GAS execution timeout 6 min | Long Claude calls can fail | 90s client timeout on `action=generate` |
| ICS feeds may be behind CORS | Raw .ics fetch fails from browser | Fetched server-side via GAS `UrlFetchApp` — no CORS restriction (AD-10) |
| ICS UTC vs local time | Events appear 4-6 hrs off | Detect `Z` suffix, parse as UTC, JS converts to local (ISSUE-003, fixed) |
| Settings sheet is world-readable | Anthropic key must never land there | `saveSettings()` deletes any key field before writing (AD-11) |
| Inline `on*=""` handlers run in a `with(document)` scope | Bare identifiers matching DOM built-ins (`children`, `removeChild`, etc.) silently resolve to the wrong object | Avoid those names for globals referenced inline; see AD-13 |
| Rolling history sheet merge | Stale tags can persist if merge favors old rows | Fresh scan data always wins on a `date|subject` key collision (AD-14) |
| localStorage only in browser | Settings lost if cleared, doesn't sync across browsers | GAS-backed Settings sheet sync + "Load Settings from Cloud" (AD-11) |
| To-Do list is localStorage-only | Doesn't sync across browsers/devices, unlike Settings | Deliberate scope cut this session (AD-18) — revisit if cross-device access becomes a real need |
| Full rolling email history intentionally includes already-passed events | Without a rule, old context reads as current in the briefing | Explicit todayISO + no-stale-content prompt rule, plus a dateISO client-side backstop (AD-16) |

---

## GAS Deployment Notes

- **Deploy type:** Web App
- **Execute as:** Me (the Google account owner)
- **Access:** Anyone (no auth required — URL is the security)
- **Versioning:** Every change requires "New version" in Manage Deployments; URL stays the same. Saving the script alone does NOT update the live endpoint.
- **Triggers:** `installDailyTrigger()` sets 6 AM daily. Run once only.
- **Logging:** `Logger.log()` visible in Executions tab

---

## Session Notes

> Add notes here after each session about architecture changes.

- **Session 1 (Mar 2026):** Architecture documented from working Sports HQ MVP.
- **Session — 2026-08-17:** Rewritten to reflect the Unified dashboard (PD-03), Child→Activity Settings model (AD-12), GAS-side settings sync (AD-11) and ICS fetch (AD-10), and childIds tagging in the Claude schema (PD-04). Retired the old three-proxy ICS waterfall and separate School HQ sections since neither reflects the live system anymore.
- **Session — 2026-08-17 (later session):** Filter architecture updated to the Category→Child→Activity cascade (PD-05), replacing the old per-calendar-name/per-sender filtering described here previously. Settings Data Model updated for global daysBack/daysForward (AD-15). Calendar Events Compact Format and the Claude Prompt Schema both updated to add dateISO (AD-16, AD-17). Added a new To-Do data flow section (AD-18), entirely separate from the briefing's own regenerated `actions` array.
- **Session — 2026-08-18:** GAS Configuration section rewritten for Script Properties, not hardcoded consts (AD-21), following a leaked API key (ISSUE-017). Added GroupMe as a new data source (AD-20): a new 3b data flow, a `groupmeGroups` line in the Settings Data Model, and two new File Inventory rows for the drafted-but-uncommitted PWA files (`manifest.json`, `sw.js`, AD-22).
- **Session — 2026-08-19:** Major rewrite of the calendar data flow — replaced the `.ics`-sourced `?cal=` GET param pipeline (Data Flow #2) with a new server-side Google Calendar scan (Data Flow #2b, AD-23), and marked Data Flow #3 (Calendar Loading) as now feeding only the Calendars tab's own week view, decoupled from the briefing. Updated the Claude Prompt Schema and system diagram to add `activityId` (PD-06), extending the Category→Child→Activity filter cascade to fully cover Summary for the first time. Updated the Calendar Events API contract section to reflect it's now a server-side-only data shape, not a GET param.
- **Session — 2026-08-21:** Confirmed the AD-23 Google Calendar scan
  working end-to-end via the full testing checklist; found and
  root-caused a real calendar-matching failure (a personal calendar
  rename doesn't change what CalendarApp reads for a calendar you
  don't own). Designed, but did not yet deploy: a `calMatchName` field
  splitting calendar-matching from display name (AD-25), Activity
  archiving in place of deletion (AD-26), a per-deployment family name
  field (AD-27), and closing the gap where the 6 AM trigger skipped
  the calendar scan entirely (AD-28).
- **Session — 2026-08-26:** Migrated the Calendars tab off the `.ics` subscription pipeline onto scanned Google Calendar data via a new `action=calendarEvents` endpoint (AD-29), rewriting Data Flow #3 to match Email History's load-once/cache-client-side pattern and marking the old `.ics` flow and its `action=ics` endpoint as retired-but-left-in-place dead code. Documented the Sheets Date-object auto-coercion gotcha (ISSUE-025) inline on the new endpoint. Typography and responsive-nav changes from this session (Quicksand/Nunito Sans, DD-04; mobile bottom nav, PD-09) are UI-only and not reflected here, since this file's scope is backend/data-flow architecture, not visual design.
- **Session — 2026-08-27:** Split `action=generate` into 4 separately-
  callable, independently-retryable steps (AD-30) — rewrote Data Flow #2
  and the API Contracts section accordingly, marked the original combined
  endpoint retired-but-kept (same treatment as `action=ics` got last
  session). Corrected Data Flow #1 — `runDailyBriefing()` was documented
  as NOT calling `scanGoogleCalendar()` (AD-28, "not yet deployed"), but
  direct inspection this session found it already does, apparently
  shipped back on 2026-08-21 (git `cbdc0b6`) and never confirmed in either
  this file or DecisionLog; DecisionLog AD-25/26/27/28 all got the same
  correction, see that file. Updated Data Flow #5 (Settings Sync) and #6
  (To-Do List) for two PD-15 changes: To-Do items gained a `dueDate` field
  (opaque to the backend, no schema change) and "Load Settings from
  Cloud"/the auto-pull-on-load path now also pull To-Dos (ISSUE-034).
  Noted AD-31's `detectApp()` change (longest-match across From+Subject,
  not first-match across From only) on the `action=scanEmail` endpoint.
  Visual-only changes from this session (accent color system PD-13, Email
  History/To-Do redesigns PD-14/15's non-data-model parts) are not
  reflected here, same scope boundary as last session's typography note.
