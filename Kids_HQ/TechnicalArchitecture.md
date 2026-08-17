# HQ Dashboard — Technical Architecture

**Last Updated:** August 2026  
**Status:** Unified Sports + School dashboard live, with a Child → Activity linking model. Superseding rationale for anything that changed lives in `DecisionLog.md` (AD-10 through AD-14, PD-03, PD-04) — this doc describes the *current* system; check the Decision Log for *why* it changed.

---

## System Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│  Browser (sports-dashboard.html) — single file, no build step        │
│                                                                        │
│  ┌───────────┐ ┌───────────┐ ┌────────────┐ ┌────────┐ ┌───────────┐ │
│  │ Summary   │ │ Calendars │ │ Email      │ │ Manual │ │ Settings  │ │
│  │           │ │           │ │ History    │ │Updates │ │           │ │
│  │ Fetches   │ │ Sends ICS │ │ GET ?action│ │ (dead  │ │ Children →│ │
│  │ stored    │ │ URL to    │ │ =history   │ │ input, │ │ Activities│ │
│  │ JSON, or  │ │ GAS via   │ │ &cat=      │ │ see    │ │ → senders │ │
│  │ ?action=  │ │ ?action=  │ │            │ │ISSUE-  │ │ / Calendars│ │
│  │ generate  │ │ ics&url=  │ │            │ │  001)  │ │           │ │
│  └───────────┘ └───────────┘ └────────────┘ └────────┘ └───────────┘ │
│                                                                        │
│  Global Child filter pill row (All / per-child) sits above the       │
│  Sports/School switch — filters Calendars, Email History, and        │
│  Summary (via childIds tagging — see PD-04)                          │
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
│    1. Save cal events from ?cal= param to the category's Calendar    │
│       Events sheet                                                    │
│    2. scanEmails(category) → Gmail search across ALL that category's │
│       Activities' senders → per-category Emails sheet, tagged with   │
│       each row's ActivityId                                          │
│    3. updateHistorySheet(category) → rolling deduped log, keyed by   │
│       date|subject, where the CURRENT scan's tag always wins on a    │
│       collision (AD-14 — this used to be a bug)                      │
│    4. generateSummaryWithClaude(category) → merges history + cal     │
│       events, resolves "[Child — Activity]" labels for context, adds │
│       a "KNOWN CHILDREN (id = name)" legend, asks Claude to tag each  │
│       card/timeline/action with childIds → POST to Claude → parse    │
│       JSON → storeSummary()                                          │
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

There is no separate `school-dashboard.html` / `SchoolEmailScanner.gs` — see Decision Log PD-03 for why that plan was reversed.

---

## Settings Data Model (Decision Log AD-12)

```
children: [
  { id, name, color }
]

activities: [
  {
    id, category: 'sports'|'school', name,
    childIds: [id, ...],           // 0, 1, or many — siblings can share a team/school
    color, daysBack, daysForward,
    senders: [ { match, app, color } ]
  }
]

cals: [
  { name, url, color, activityId }  // '' = unlinked
]

maxEmails: 100
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
      scanEmails(category) → Gmail → {Cat}Emails sheet + {Cat}EmailHistory sheet
      generateSummaryWithClaude(category)
        → read {Cat}EmailHistory sheet (merged with this scan, fresh data wins — AD-14)
        → read {Cat}CalendarEvents sheet (last pushed from dashboard)
        → build category-specific prompt, with a KNOWN CHILDREN legend
        → POST to Claude API
        → storeSummary(category) → {Cat}Summary sheet
```

### 2. On-Demand Refresh from Dashboard
```
User clicks Refresh Briefing → generate() in JS
  → calEventsParam() → encodes up to 60 calendar events (current category, current
    child filter does NOT narrow this — it's the full category's events) as compact JSON
  → fetch GET /exec?action=generate&cat={active}&cal=[JSON]
  → GAS: save cal events → scan Gmail → call Claude → store
  → return { summary, generatedAt } → renderOutput() → cached as lastBriefingData
    → renderBriefingFiltered() applies the active Child filter client-side
```

### 3. Calendar Loading
```
User clicks Load Calendars
  → for each configured calendar row (name, url, color, activityId):
      normalizeIcsUrl() → fix teamsnap http/webcal/google embed URLs
      fetchIcsViaGas() → GET /exec?action=ics&url=... (server-side fetch, AD-10)
      parseICS() → split by VEVENT → parse DTSTART/SUMMARY/LOCATION
      each parsed event is tagged with that calendar's activityId
  → loadedEvents[] in memory (stale until the next Load Calendars click —
    editing a calendar's Activity link in Settings does NOT retroactively
    update already-loaded events; re-click Load Calendars after relinking)
  → rebuildTeamFilters() → renderCalendar(), which applies category +
    child filters together (categoryFilteredEvents() → filteredEvents())
  → [on next Refresh Briefing] → calEventsParam() sends events to GAS
```

### 4. Email History Browsing
```
User opens Email History tab
  → loadHistory() → fetch GET /exec?action=history&cat={active}
  → GAS reads {Cat}EmailHistory sheet → returns JSON array (each row incl. activityId)
  → renderHistory() → filters by selected app AND (if set) by the active Child,
    resolving activityId → Activity → childIds
```

### 5. Settings Sync
```
Local edit (add/edit/remove Child, Activity, sender, or Calendar)
  → saveSettingsLocal() → localStorage['gdhq_settings']
  → (only on explicit Save) pushSettingsToGas() → GET /exec?action=saveSettings&settings=[JSON]
      → GAS strips any API key field, writes to the Settings sheet

Fresh browser, GAS URL already known, no local settings
  → auto-triggers pullSettingsFromGas(silent) once on load

Manual "☁ Load Settings from Cloud" button
  → pullSettingsFromGas(false) → same fetch, shows a status message either way
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
    "timeline": [{ "date": "Mon Mar 10", "time": "3:30 PM", "event": "", "childIds": ["..."] }],
    "actions": [{ "text": "string", "childIds": ["..."] }]
  },
  "generatedAt": "ISO 8601 string"
}
```
Note: `actions` were plain strings before the child-tagging change; the dashboard's `renderBriefingFiltered()` accepts both shapes so old cached briefings still render.

**GET /exec?action=generate&cat=&cal=JSON** — Regenerates that category's briefing, returns same shape

**GET /exec?action=history&cat=[&app=AppName]** — Returns email history
```json
{ "emails": [{ "date": "ISO", "from": "", "app": "", "subject": "", "body": "", "activityId": "" }] }
```

**GET /exec?action=ics&url=...** — Fetches one `.ics` feed server-side, category-agnostic
```json
{ "ok": true, "text": "raw ICS content" }
```

**GET /exec?action=getSettings** — Returns the Settings sheet's JSON (never includes the Anthropic key)

**GET /exec?action=saveSettings&settings=JSON** — Writes settings (strips any Anthropic key field first)

**POST /exec** — Legacy calendar-events endpoint, retained but unused by the current dashboard

### Calendar Events Compact Format (GET param, sent TO GAS from the dashboard)
```json
[{ "t": "team", "s": "summary", "d": "Mon Mar 10", "h": "3:30 PM", "l": "location" }]
```
Capped at 60 events, max 4000 chars total. Skipped if over limit.

### Claude Prompt Schema
Claude is prompted (per category, via `buildSportsPrompt`/`buildSchoolPrompt`) to return ONLY valid JSON:
```json
{
  "priority": "string",
  "cards": [{ "childIds": ["..."], "...": "..." }],
  "timeline": [{ "childIds": ["..."], "...": "..." }],
  "actions": [{ "text": "...", "childIds": ["..."] }]
}
```
Tag values (sports): `urgent` | `today` | `soon` | `info`. Tag values (school): `urgent` | `today` | `soon` | `exam` | `info`.  
`childIds` must use only the exact ids from the `KNOWN CHILDREN (id = name)` legend given in the prompt; an empty array means general/unclear, and the dashboard treats that as "show under every filter."

---

## Key Technical Constraints

| Constraint | Impact | Solution |
|-----------|--------|----------|
| CORS blocks browser → Claude API | Can't call Claude from browser | All AI calls go through GAS (AD-01) |
| CORS blocks browser → GAS (sometimes) | Fetch failures | allorigins.win proxy fallback, `/exec` only |
| GAS GET URL length limit ~8kb | Can't send unlimited calendar data | Cap events at 60, short keys, 4000 char limit |
| GAS execution timeout 6 min | Long Claude calls can fail | 90s client timeout on `action=generate` |
| ICS feeds may be behind CORS | Raw .ics fetch fails from browser | Fetched server-side via GAS `UrlFetchApp` — no CORS restriction (AD-10) |
| ICS UTC vs local time | Events appear 4-6 hrs off | Detect `Z` suffix, parse as UTC, JS converts to local (ISSUE-003, fixed) |
| Settings sheet is world-readable | Anthropic key must never land there | `saveSettings()` deletes any key field before writing (AD-11) |
| Inline `on*=""` handlers run in a `with(document)` scope | Bare identifiers matching DOM built-ins (`children`, `removeChild`, etc.) silently resolve to the wrong object | Avoid those names for globals referenced inline; see AD-13 |
| Rolling history sheet merge | Stale tags can persist if merge favors old rows | Fresh scan data always wins on a `date|subject` key collision (AD-14) |
| localStorage only in browser | Settings lost if cleared, doesn't sync across browsers | GAS-backed Settings sheet sync + "Load Settings from Cloud" (AD-11) |

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
