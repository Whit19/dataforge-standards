# HQ Dashboard — Technical Architecture

**Last Updated:** March 2026  
**Status:** Sports HQ MVP complete. School HQ architecture in design.

---

## System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  Browser (sports-dashboard.html)                                │
│                                                                 │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │ Summary Tab │  │ Calendar Tab │  │ Email History Tab      │ │
│  │             │  │              │  │                        │ │
│  │ Fetches     │  │ Fetches .ics │  │ GET /exec?action=      │ │
│  │ stored JSON │  │ via proxy    │  │ history → email log    │ │
│  │ from GAS    │  │ Parses ICS   │  │                        │ │
│  │             │  │ Sends events │  │                        │ │
│  │ OR triggers │  │ to GAS via   │  │                        │ │
│  │ regeneration│  │ ?cal= param  │  │                        │ │
│  └─────────────┘  └──────────────┘  └────────────────────────┘ │
│                                                                 │
│  Settings stored in: localStorage (GAS URL + calendar URLs)    │
└──────────────────────────────┬──────────────────────────────────┘
                               │ fetch() via CORS proxy fallback
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  Google Apps Script (SportsEmailScanner.gs)                    │
│  Deployed as Web App (Execute as: Me, Access: Anyone)          │
│                                                                 │
│  doGet(?action=generate)                                        │
│    1. Save cal events from ?cal= param to CalendarEvents sheet │
│    2. scanSportsEmails() → Gmail search → SportsEmails sheet   │
│    3. updateHistorySheet() → rolling 15-day deduped log        │
│    4. generateSummaryWithClaude() → merge emails + cal events  │
│       → POST to api.anthropic.com/v1/messages                  │
│       → parse JSON response → storeSummary()                   │
│                                                                 │
│  doGet(?action=history)                                         │
│    → Return EmailHistory sheet as JSON                         │
│                                                                 │
│  doGet() default                                                │
│    → Return Summary sheet (generated_at + summary_json)        │
│                                                                 │
│  doPost()                                                       │
│    → Receive calendar events (legacy endpoint, still active)   │
│                                                                 │
│  Daily trigger: runDailyBriefing() at 6 AM                     │
└──────────────────────────────┬──────────────────────────────────┘
                               │
          ┌────────────────────┼───────────────────────┐
          ▼                    ▼                       ▼
 ┌─────────────────┐  ┌──────────────────┐   ┌────────────────────┐
 │  Gmail API      │  │ Anthropic API    │   │ Google Sheets      │
 │  (via GAS       │  │ claude-sonnet-   │   │ "Game Day HQ"      │
 │  GmailApp)      │  │ 4-5              │   │                    │
 │                 │  │ max_tokens: 4000 │   │ Tabs:              │
 │  Query: senders │  │ Returns JSON     │   │ - SportsEmails     │
 │  + date range   │  │ briefing object  │   │ - EmailHistory     │
 └─────────────────┘  └──────────────────┘   │ - Summary          │
                                             │ - CalendarEvents   │
                                             └────────────────────┘
```

---

## File Inventory

| File | Type | Role | Status |
|------|------|------|--------|
| `sports-dashboard.html` | Single-file HTML/CSS/JS | Frontend dashboard | ✅ Live |
| `SportsEmailScanner.gs` | Google Apps Script | Backend / AI orchestration | ✅ Live |
| `school-dashboard.html` | Single-file HTML/CSS/JS | School HQ frontend | 🔲 Planned |
| `SchoolEmailScanner.gs` | Google Apps Script | School HQ backend | 🔲 Planned |

---

## Data Flows

### 1. Daily Auto-Briefing (6 AM)
```
GAS trigger → runDailyBriefing()
  → scanSportsEmails() → Gmail → SportsEmails sheet + EmailHistory sheet
  → generateSummaryWithClaude()
    → read EmailHistory sheet (15 days)
    → read CalendarEvents sheet (last pushed from dashboard)
    → build prompt → POST to Claude API
    → storeSummary() → Summary sheet
```

### 2. On-Demand Refresh from Dashboard
```
User clicks Refresh → generate() in JS
  → calEventsParam() → encodes up to 60 calendar events as compact JSON
  → fetch GET /exec?action=generate&cal=[JSON]
  → GAS: save cal events → scan Gmail → call Claude → store
  → return { summary, generatedAt } → renderOutput() in JS
```

### 3. Calendar Loading
```
User clicks Load Calendars
  → for each configured .ics URL:
      normalizeIcsUrl() → fix teamsnap http/webcal/google embed URLs
      fetchViaProxy() → try allorigins → corsproxy.io → codetabs
      parseICS() → split by VEVENT → parse DTSTART/SUMMARY/LOCATION
  → loadedEvents[] in memory
  → rebuildTeamFilters() → renderCalendar()
  → [on next Refresh] → calEventsParam() sends events to GAS
```

### 4. Email History Browsing
```
User opens Email History tab
  → loadHistory() → fetch GET /exec?action=history
  → GAS reads EmailHistory sheet → returns JSON array
  → renderHistory() → accordion list grouped by app
```

---

## API Contracts

### GAS Web App Endpoints

**GET /exec** — Returns stored briefing
```json
{
  "summary": {
    "priority": "string",
    "cards": [{ "source": "", "sourceColor": "#hex", "title": "", "body": "", "bullets": [], "tag": "urgent|today|soon|info" }],
    "timeline": [{ "date": "Mon Mar 10", "time": "3:30 PM", "event": "" }],
    "actions": ["string"]
  },
  "generatedAt": "ISO 8601 string"
}
```

**GET /exec?action=generate[&cal=JSON]** — Regenerates briefing, returns same shape

**GET /exec?action=history[&app=AppName]** — Returns email history
```json
{
  "emails": [{ "date": "ISO", "from": "", "app": "", "subject": "", "body": "" }]
}
```

**POST /exec** — Accepts calendar events
```json
{ "events": [{ "team": "", "date": "", "time": "", "summary": "", "location": "" }] }
```

### Calendar Events Compact Format (GET param)
```json
[{ "t": "team", "s": "summary", "d": "Mon Mar 10", "h": "3:30 PM", "l": "location" }]
```
Capped at 60 events, max 4000 chars total. Skipped if over limit.

### Claude Prompt Schema
Claude is prompted to return ONLY valid JSON (no markdown):
```json
{
  "priority": "string",
  "cards": [...],
  "timeline": [...],
  "actions": [...]
}
```
Tag values: `urgent` | `today` | `soon` | `info`

---

## Key Technical Constraints

| Constraint | Impact | Solution |
|-----------|--------|----------|
| CORS blocks browser → Claude API | Can't call Claude from browser | All AI calls go through GAS |
| CORS blocks browser → GAS (sometimes) | Fetch failures | allorigins.win proxy fallback |
| GAS GET URL length limit ~8kb | Can't send unlimited calendar data | Cap events at 60, short keys, 4000 char limit |
| GAS execution timeout 6 min | Long Claude calls can fail | 90s client timeout, async generation |
| ICS feeds may be behind CORS | Raw .ics fetch fails from browser | Three-proxy waterfall (allorigins → corsproxy.io → codetabs) |
| ICS UTC vs local time | Events appear 4-6 hrs off | Detect `Z` suffix, parse as UTC, JS converts to local |
| localStorage only in browser | Settings lost if cleared | Users paste URLs into Settings; consider export/import |

---

## CORS Proxy Stack (Calendar Fetching)

Tried in order; first success wins:

1. `https://api.allorigins.win/get?url={encoded}`  — Returns `{ contents: "..." }`
2. `https://corsproxy.io/?{encoded}` — Returns raw text
3. `https://api.codetabs.com/v1/proxy?quest={encoded}` — Returns raw text

All have 8-second abort timeout. If all fail, error surfaced to user.

---

## GAS Deployment Notes

- **Deploy type:** Web App
- **Execute as:** Me (the Google account owner)
- **Access:** Anyone (no auth required — URL is the security)
- **Versioning:** Every change requires "New version" in Manage Deployments; URL stays the same
- **Triggers:** `installDailyTrigger()` sets 6 AM daily. Run once only.
- **Logging:** `Logger.log()` visible in Executions tab

---

## School HQ — Planned Architecture

Will mirror Sports HQ exactly:

```
school-dashboard.html  ←→  SchoolEmailScanner.gs  ←→  Claude API
                                    ↓
                           Google Sheet "School HQ"
                           Tabs: SchoolEmails, SchoolHistory,
                                 SchoolSummary, SchoolCalendar
```

Differences from Sports:
- Sender domains: school district domains, teacher addresses, PTA lists
- Claude prompt tuned for: assignments, tests, deadlines, school events
- Cards color-coded by subject (Math, Science, English, etc.)
- Timeline shows "due dates" rather than game times
- Tag meanings: `urgent` = due today/tomorrow, `soon` = this week, `exam` = test/quiz, `info` = FYI

---

## Session Notes

> Add notes here after each session about architecture changes.

- **Session 1 (Mar 2026):** Architecture documented from working Sports HQ MVP.

