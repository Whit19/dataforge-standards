# Game Day HQ — Project Session Starter
**Protocol:** Load MASTER_CLAUDE_PROTOCOL.md before this file.
Repo: github.com/Whit19/dataforge-standards
## What This Is
A personal sports dashboard for managing multiple kids' sports teams. It aggregates emails from 7+ sports apps, syncs team calendars, and uses Claude AI (via Google Apps Script) to generate a daily 7-day briefing with a priority, app cards, schedule timeline, and action items.

---

## Two Output Files

### 1. `sports-dashboard.html`
A single-file HTML dashboard saved locally or hosted (e.g. Netlify Drop). No backend — all logic is in-browser JS + calls to the GAS Web App.

### 2. `SportsEmailScanner.gs`
Google Apps Script bound to a Google Sheet ("Game Day HQ"). Deployed as a Web App (Execute as: Me, Access: Anyone). This is the backend — it scans Gmail, calls the Anthropic Claude API, stores results, and serves them to the dashboard.

---

## Architecture

```
Browser (sports-dashboard.html)
  │
  ├── Loads .ics calendar feeds via CORS proxies (allorigins, corsproxy.io, codetabs)
  ├── On page load: GET /exec → fetches last stored summary from GAS
  ├── On Refresh: GET /exec?action=generate&cal=[events JSON] → triggers GAS to regenerate
  └── On Email History tab: GET /exec?action=history → fetches 15-day email log

Google Apps Script (SportsEmailScanner.gs)
  │
  ├── doGet(?action=generate) → scans Gmail + saves cal events + calls Claude + stores summary
  ├── doGet(?action=history)  → returns EmailHistory sheet as JSON
  ├── doGet() default         → returns stored Summary sheet as JSON
  ├── doPost()                → receives calendar events (legacy, now handled via GET param)
  └── Daily trigger at 6 AM  → runs runDailyBriefing() automatically
```

---

## Google Sheet Structure
The GAS script creates/uses a Google Sheet named **"Game Day HQ"** with these tabs:
- **SportsEmails** — latest scan results (last 15 days)
- **EmailHistory** — rolling 15-day deduped log of all sports emails
- **Summary** — stores `generated_at` and `summary_json` (Claude's output)
- **CalendarEvents** — events pushed from dashboard (next 14 days, all teams)

---

## GAS Configuration (top of script)

```javascript
const ANTHROPIC_API_KEY = 'sk-ant-...'; // your key from console.anthropic.com

const SPORTS_SENDERS = [
  'teamsnap.com', 'email.teamsnap.com',
  'sportsengine.com', 'ngin.com',
  'gamechanger.com', 'gc.com',
  'groupme.com',
  'playmetrics.com',
  'leagueapps.com', 'leagueapps.io',
  'carpool-kids.com',
  'demosphere.com', 'activity.com',
  // Specific coach/team addresses:
  'kcubb12@gmail.com',
  'juliemclarenlax@gmail.com',
  'rob.dubinski@wfbschools.com',
  'g.barlow94@gmail.com',
  'omalley@muhs.edu',
];

const DAYS_BACK = 15;       // how far back to scan Gmail
const DAYS_FORWARD = 7;     // briefing horizon
const MAX_EMAILS = 100;
```

---

## Dashboard Tabs
| Tab | Purpose |
|-----|---------|
| ⚡ Summary | Home screen — shows last briefing, Refresh button auto-loads on open |
| 📅 Calendars | Weekly list view, navigate by week, filter by team, Load Calendars button |
| 📬 Email History | 15-day email log filtered by app (TeamSnap, Coach/Team, etc.) |
| ✏️ Manual Updates | Paste text from any app as fallback input to the briefing |
| ⚙ Settings | Web App URL + team calendar .ics subscriptions (saved to localStorage) |

---

## Calendar URLs (saved in browser localStorage)
- **Black Lax**: `https://api.leagueapps.io/api/mobile/v1/v3/public/events/schedule/ical?includeUserId=50794025`
- **WNS Lax**: Google Calendar embed URL (auto-converted to .ics in dashboard)
- **MUHS Lax**: `https://ical-cdn.teamsnap.com/team_schedule/f43ea567-4f2b-4fe3-a773-7eca80ab02ee.ics`
- **Bavs**: `https://calendar.playmetrics.com/calendars/c219/t380069/p0/t679051E0/f/calendar.ics`
- **Fighting Sue**: LeagueApps .ics feed (UTC timezone — fixed in parser)

---

## Key Technical Decisions & Fixes Made This Session

### Claude API lives in GAS, not the browser
- Browser → Claude direct calls are blocked by CORS
- GAS uses `UrlFetchApp` which has no CORS restrictions
- Model: `claude-sonnet-4-5`, `max_tokens: 4000`

### Calendar events sent via GET param (not POST)
- POST from browser to GAS was CORS-blocked
- Solution: encode up to 60 events as compact JSON in `?cal=` URL param
- Short keys (`t/s/d/h/l`) keep the param under 4000 chars
- GAS reads param, saves to CalendarEvents sheet, passes to Claude

### ICS timezone fix
- Events with `Z` suffix (UTC) were being parsed as local time → 4hr offset
- Fix: detect `Z` suffix and parse as true UTC; JS auto-converts to local time

### Briefing uses full 15-day email history
- `generateSummaryWithClaude()` now merges latest scan rows with full EmailHistory sheet
- Ensures all senders/teams are represented, not just the last 2-day window

### fetchFromGas timeouts
- Default fetches: 15 second timeout
- `action=generate` fetches: 90 second timeout (Claude call takes 20-60s)
- Proxy fallback (allorigins.win) skips `cal=` param to avoid URL-too-long errors

---

## Deployment Workflow (after any GAS change)
1. Paste updated script into Extensions → Apps Script
2. Save (Ctrl+S)
3. Deploy → Manage Deployments → ✏️ edit → New version → Deploy
4. **URL does not change** — no dashboard update needed
5. No need to re-run `installDailyTrigger` or `runDailyBriefing` unless testing

## First-Time Setup (for new environment)
1. Get Anthropic API key at console.anthropic.com → add billing credits
2. Open Google Sheet → Extensions → Apps Script → paste `SportsEmailScanner.gs`
3. Add API key at top of script
4. Run `installDailyTrigger` once (sets 6 AM daily schedule)
5. Run `runDailyBriefing` to test — check Execution Log
6. Deploy as Web App → Execute as: Me, Access: Anyone
7. Open dashboard → Settings tab → paste Web App URL
8. Calendars tab → Load Calendars
9. Summary tab → Refresh Briefing

---

## Possible Next Features
- GroupMe API integration (direct message pull, no email delay)
- Push notifications / SMS for same-day urgent events
- Weather forecast injected into game day events
- Carpool coordination view
