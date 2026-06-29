# HQ Dashboard — Best Methods

**Last Updated:** June 2026  
**Purpose:** Hard-won lessons learned during development. Read this before writing any code for this project. Prevents re-making known mistakes.

---

## How to Use This File

Add entries any time a bug is fixed, a workaround is discovered, or a "gotcha" is encountered. Each entry should explain *what* to do and *why* — not just the solution but the failure mode it prevents.

---

## Google Apps Script (GAS)

---

### BM-01 — Always Use `muteHttpExceptions: true` in `UrlFetchApp`

**Rule:** Every `UrlFetchApp.fetch()` call must include `muteHttpExceptions: true`.

**Why:** Without it, any non-200 HTTP response throws an exception and you lose the error body. With it, you get back the full response including the error message from the API. This is essential for diagnosing Claude API errors (400 = bad model name, 401 = bad API key, 429 = rate limit).

```javascript
const resp = UrlFetchApp.fetch(url, {
  method: 'post',
  headers: { ... },
  payload: JSON.stringify({ ... }),
  muteHttpExceptions: true  // ← always include this
});
const code = resp.getResponseCode();
const text = resp.getContentText();
if (code !== 200) {
  Logger.log('Error body: ' + text); // now you can see what went wrong
}
```

---

### BM-02 — GAS Code Changes Require a New Deployment Version

**Rule:** After editing `SportsEmailScanner.gs`, always deploy as a **New Version** under Manage Deployments. Editing the script and saving is not enough.

**Why:** GAS Web Apps serve from a pinned deployment snapshot, not from the live editor. Changes to the script file don't take effect until you increment the deployment version. The URL stays the same — no dashboard update is needed.

**Steps:**
1. Make change → Save (Ctrl+S)
2. Deploy → Manage Deployments → ✏️ (edit) → Version: New Version → Deploy
3. Done. Same URL, new code.

**Symptom if forgotten:** Code changes have no effect; old behavior persists. Confusing to debug.

---

### BM-03 — Use `getSpreadsheet()` Helper, Not `SpreadsheetApp.getActiveSpreadsheet()` Directly

**Rule:** Always use the `getSpreadsheet()` helper function, which includes a `DriveApp.getFilesByName()` fallback.

**Why:** `getActiveSpreadsheet()` only works when the script is bound to a spreadsheet and triggered from within that spreadsheet context. When the script runs via Web App (triggered from a URL), there may be no "active" spreadsheet. The fallback searches by name.

```javascript
function getSpreadsheet() {
  try {
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    if (ss) return ss;
  } catch(e) {}
  const files = DriveApp.getFilesByName('Game Day HQ');
  if (files.hasNext()) return SpreadsheetApp.open(files.next());
  // create if needed
}
```

---

### BM-04 — 400 Error from Claude API? Check the Model Name First

**Rule:** When Claude API returns a 400 error, check the `model` string before anything else.

**Why:** The most common cause of 400 errors from the Anthropic API is an invalid or misspelled model name. Current valid model: `claude-sonnet-4-5`. Model names change with new releases. Always copy the model string from the Anthropic docs, don't type it from memory.

**Diagnosis order:**
1. `code === 400` → check model name
2. `code === 401` → bad API key
3. `code === 429` → rate limit / quota
4. `code === 500` → Anthropic outage

---

### BM-05 — Strip Markdown Fences Before Parsing Claude's JSON Response

**Rule:** Always strip ` ```json ` and ` ``` ` fences from Claude's response before calling `JSON.parse()`.

**Why:** Despite being prompted to return "ONLY valid JSON (no markdown, no backticks)", Claude occasionally wraps responses in markdown fences. This causes `JSON.parse()` to throw. The strip step is cheap insurance.

```javascript
const raw = (apiData.content || []).map(c => c.text || '').join('');
const summary = JSON.parse(raw.replace(/```json|```/g, '').trim());
```

---

## Calendar / ICS Parsing

---

### BM-06 — Always Detect UTC (`Z`) Suffix in ICS DTSTART Fields

**Rule:** In `parseICS()`, always check if the DTSTART value ends with `Z` and parse it as UTC, not local time.

**Why:** Some providers (LeagueApps, others) use UTC timestamps with `Z` suffix. Parsing these as local time causes a 5-6 hour offset (CDT/CST). JavaScript's `Date` constructor handles UTC correctly if the `Z` is preserved.

```javascript
if (dtRaw.endsWith('Z')) {
  dt = new Date(dtRaw.replace(/(\d{4})(\d{2})(\d{2})T(\d{2})(\d{2})(\d{2})Z/, '$1-$2-$3T$4:$5:$6Z'));
} else if (dtRaw.length === 8) {
  // All-day: YYYYMMDD
  dt = new Date(dtRaw.slice(0,4) + '-' + dtRaw.slice(4,6) + '-' + dtRaw.slice(6,8) + 'T00:00:00');
} else {
  // Floating local time
  dt = new Date(dtRaw.replace(/(\d{4})(\d{2})(\d{2})T(\d{2})(\d{2})(\d{2}).*/, '$1-$2-$3T$4:$5:$6'));
}
```

**Re-test when clocks change** (CST = UTC-6, CDT = UTC-5). The parser is correct; event times will shift by 1 hour at DST transitions as expected.

---

### BM-07 — Normalize .ics URLs Before Proxying

**Rule:** Always run ICS URLs through `normalizeIcsUrl()` before passing to `fetchViaProxy()`.

**Why:** Three common URL problems to catch:
1. `webcal://` → replace with `https://`
2. Google Calendar HTML embed URLs → convert to `.ics` format
3. TeamSnap `http://ical-cdn.teamsnap.com` → upgrade to `https://`

Without normalization, proxies will either reject the URL or return HTML instead of ICS data.

---

### BM-08 — Proxy Waterfall: allorigins → corsproxy.io → codetabs

**Rule:** Always try proxies in this order. Don't skip to a later proxy unless the earlier one returns a non-ICS response.

**Why:** allorigins.win has the best reliability but caches aggressively. corsproxy.io is less cached but less stable. codetabs is the most reliable fallback but slowest. The waterfall balances freshness vs. resilience.

**Known limitation:** allorigins may serve stale ICS data (up to a few hours old). If a user reports missing recent events, try adding a cache-busting query param to the ICS URL: `?_cb=${Date.now()}`.

---

## Browser / Frontend

---

### BM-09 — `action=generate` Needs a 90-Second Timeout, Not 15

**Rule:** Always use a 90-second client-side timeout for `action=generate` fetches. Use 15 seconds for all other GAS calls.

**Why:** The generate call triggers: Gmail scan + EmailHistory merge + Claude API call. The Claude call alone can take 20-60 seconds. 15 seconds will time out before the response arrives.

```javascript
const timeout = isGenerate ? 90000 : 15000;
fetch(url, { signal: AbortSignal.timeout(timeout) });
```

---

### BM-10 — Cap Calendar Events at 60 and 4000 Chars for GET Param

**Rule:** When encoding calendar events as a `?cal=` GET param, cap at 60 events AND check that the JSON string is under 4000 chars. Skip the param entirely if over the limit.

**Why:** GAS Web App URLs have practical limits around 8kb. Encoded JSON can grow quickly with many teams and events. The 60-event / 4000-char cap keeps the URL safely under limits. Use short keys (`t/s/d/h/l` instead of `team/summary/date/time/location`) to maximize the number of events that fit.

---

### BM-11 — `historyLoaded` Flag Must Be Reset After `generate()`

**Rule:** After a successful `generate()` call, set `historyLoaded = false` so the Email History tab re-fetches on next visit.

**Why:** The `historyLoaded` flag prevents re-fetching on every tab visit (good). But after a refresh that rescans Gmail, the history data is stale. Without the reset, users see old history until they manually reload the page.

```javascript
async function generate() {
  // ... generate logic ...
  historyLoaded = false; // reset so Email History re-fetches
}
```

**Status:** This is ISSUE-006 — a known one-line fix not yet applied.

---

### BM-12 — Proxy Fallback in `fetchFromGas` Should Omit the `cal=` Param

**Rule:** When the direct GAS fetch fails and you fall back to the allorigins.win proxy, use only `?action=generate` — not the full URL with `cal=` param.

**Why:** The allorigins proxy has its own URL length limits. A long `cal=` param appended to an already-encoded URL can cause the proxy to reject or truncate the request. Calendar events were already sent to GAS in a recent generate call and are stored in the CalendarEvents sheet, so omitting the param is acceptable.

---

## Data & Prompt Engineering

---

### BM-13 — Manual Updates Must Be Explicitly Added to the Claude Prompt

**Rule:** If the Manual Updates tab is used, its textarea values must be read in `generate()`, encoded, and included in the GAS prompt as a dedicated `=== MANUAL UPDATES ===` section.

**Why:** Claude only knows what's in the prompt. Textarea content sitting in the browser is invisible to GAS. This is currently broken (ISSUE-001 — highest priority open bug).

**Proposed implementation:**
```javascript
// In generate() — collect manual updates
const manualSections = [
  { label: 'TeamSnap', id: 'e-teamsnap' },
  { label: 'SportsEngine', id: 'e-sportsengine' },
  // etc.
].map(s => ({ label: s.label, text: document.getElementById(s.id).value.trim() }))
  .filter(s => s.text.length > 0);

// Encode and send as ?manual= param (watch URL length)
```

---

### BM-14 — Briefing Should Use Full 15-Day Email History, Not Just Latest Scan

**Rule:** `generateSummaryWithClaude()` must merge the current scan rows with the full `EmailHistory` sheet before building the Claude prompt.

**Why:** The daily scan only covers emails from the last 15 days, but if the GAS runs right after midnight, many of those emails were already in the history from the previous run. More importantly, a game confirmed 10 days ago won't be in today's scan results if it's outside the immediate window. The history merge ensures all context is included.

This is already implemented — keep it. Do not "simplify" by removing the history merge.

---

## Security

---

### BM-15 — Never Put the Anthropic API Key in Any Committed File

**Rule:** The API key lives only inside the GAS script editor (not in any committed MD file, not in the HTML dashboard, not in any GitHub-tracked file).

**Why:** The GAS script is not stored in GitHub — it lives in Google Apps Script cloud. The key is safe there. Any committed file (including documentation) is public or at risk of exposure.

If you need to reference the key location in docs: write `[see GAS script top-of-file constant ANTHROPIC_API_KEY]`.

---

## Session Notes

> Append entries as new lessons are learned. Never delete entries — mark outdated ones as `[Superseded]`.

- **Session 1 (Mar 2026):** File created. BM-01 through BM-15 documented from MVP development.