# HQ Dashboard — Best Methods

**Last Updated:** 2026-08-18  
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

### BM-07 — Normalize .ics URLs Before Handing Them Off for Fetching

**Rule:** Always run ICS URLs through `normalizeIcsUrl()` before passing them along to be fetched.

**Why:** Three common URL problems to catch:
1. `webcal://` → replace with `https://`
2. Google Calendar HTML embed URLs → convert to `.ics` format
3. TeamSnap `http://ical-cdn.teamsnap.com` → upgrade to `https://`

Without normalization, the fetch will either be rejected or return HTML instead of ICS data. **[Still applies]** — this step still runs before the URL is sent to GAS's `action=ics` endpoint (see BM-08 for how the fetch itself changed).

---

### BM-08 — Proxy Waterfall: allorigins → corsproxy.io → codetabs

**Status:** `[Superseded by AD-10]` — kept here for history; do not reintroduce this pattern.

**Original rule:** Always try proxies in this order for fetching `.ics` calendar data from the browser. allorigins.win had the best reliability but cached aggressively; corsproxy.io was less cached but less stable; codetabs was the slowest but most reliable fallback.

**Why it was replaced:** This became the dashboard's biggest reliability problem in practice. A direct fetch test against each proxy showed allorigins.win timing out and corsproxy.io no longer proxying the old query format at all (returning its own marketing homepage instead of calendar data) — the API had changed out from under the integration with no warning. Calendar `.ics` fetching now happens server-side through GAS's `UrlFetchApp` (`doGet(?action=ics&url=...)`), which has no CORS restriction and doesn't depend on any third party's uptime or API stability. See DecisionLog AD-10.

**Lesson to keep even though the specific proxies are gone:** when a browser-side integration depends on a public, unauthenticated third-party proxy for something core to the app, budget for it breaking silently and prefer routing through your own server-side code (GAS, in this stack) if that's an option.

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

**Status:** Still open as of 2026-08-17 — reconfirmed by reading the current `generate()` function; the reset is still not present. This is IssuesTracker ISSUE-006, a known one-line fix not yet applied.

---

### BM-12 — Proxy Fallback in `fetchFromGas` Should Omit the `cal=` Param

**Rule:** When the direct GAS fetch fails and you fall back to the allorigins.win proxy, use only `?action=generate` — not the full URL with `cal=` param.

**Why:** The allorigins proxy has its own URL length limits. A long `cal=` param appended to an already-encoded URL can cause the proxy to reject or truncate the request. Calendar events were already sent to GAS in a recent generate call and are stored in the CalendarEvents sheet, so omitting the param is acceptable.

**Note (2026-08-17):** This allorigins fallback is unrelated to BM-08's now-removed ICS proxy waterfall — it's a separate, still-active fallback purely for reaching the `/exec` endpoint itself when a direct fetch fails. Don't confuse the two.

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

**Note (2026-08-17):** Still open, and now has an added wrinkle — the hardcoded 7-app list above (TeamSnap/SportsEngine/GameChanger/GroupMe/Playmetrics/League/Carpool Kids) predates the Child→Activity/sender model (DecisionLog AD-12) and no longer necessarily matches what the user has actually configured as senders. When this gets fixed, rebuild the tab to generate one section per configured Activity instead of a hardcoded app list — don't just wire the existing hardcoded textareas into the prompt as-is.

---

### BM-14 — Briefing Should Use Full Rolling Email History, Not Just Latest Scan

**Rule:** `generateSummaryWithClaude()` must merge the current scan rows with the full rolling `EmailHistory` sheet before building the Claude prompt.

**Why:** The daily scan only covers emails from the last N days, but if the GAS runs right after midnight, many of those emails were already in the history from the previous run. More importantly, a game confirmed 10 days ago won't be in today's scan results if it's outside the immediate window. The history merge ensures all context is included.

This is already implemented — keep it. Do not "simplify" by removing the history merge. See BM-16 below for a real bug that was hiding inside this merge.

---

## Security

---

### BM-15 — Never Put the Anthropic API Key in Any Committed File

**Rule:** The API key lives only inside the GAS script editor (not in any committed MD file, not in the HTML dashboard, not in any GitHub-tracked file).

**Why:** The GAS script is not stored in GitHub — it lives in Google Apps Script cloud. The key is safe there. Any committed file (including documentation) is public or at risk of exposure.

If you need to reference the key location in docs: write `[see GAS script top-of-file constant ANTHROPIC_API_KEY]`.

**Correction (2026-08-18):** The assumption above — "the GAS script is not stored in GitHub" — turned out to be false in practice for this project: `SportsEmailScanner.gs` was in fact git-tracked alongside `sports-dashboard.html` in a private repo, and a live `ANTHROPIC_API_KEY` was committed and pushed as a plaintext const (see IssuesTracker ISSUE-017). The durable fix is BM-22 below — never hardcode the value at all, regardless of where the file lives or how private the repo is believed to be.

**See also BM-18** for a related, newer risk: the key leaking through a *runtime data store* (a Settings sheet) rather than a *committed file*.

---

## Additions — Session 2026-08-17

Three new lessons from this session's Child→Activity Settings rebuild and the two bugs it surfaced. Numbered to continue the existing sequence (next available was BM-16); not physically inserted into the sections above, per this file's append-only convention — see the "See also" links above for where they connect to existing entries.

---

### BM-16 — Rolling History/Log Sheet Merges Must Let the Latest Scan Win, Not the First-Seen Row

**Rule:** When merging fresh scan results into a rolling, deduped sheet (e.g. `updateHistorySheet()`), key the merge by a stable identity (e.g. `date|subject`) and let the **current** scan's row overwrite any existing row with the same key — never let whichever row was written first win permanently.

**Why:** A full re-scan (like `scanEmails()`) already recomputes every field correctly for the entire lookback window on every single run — it's not incremental. If the merge instead favors "whatever's already on the sheet" on a key collision, any field that was wrong the first time a row was written (e.g. an `ActivityId` assigned before Settings was fully configured) gets stuck that way forever, silently, with no error, because that row will never look "new" again.

**Symptom if forgotten:** a feature built on top of that sheet (here: per-child email filtering) appears "half broken" in a confusing pattern — correct for anything written after a config fix, wrong for everything written before, with no obvious cause from the outside.

```javascript
const merged = new Map();
existing.forEach(r => merged.set(key(r), r));
newRows.forEach(r => merged.set(key(r), r)); // fresh scan wins on collision
```

**Where this bit us:** `updateHistorySheet()` in `SportsEmailScanner.gs`, after the Child→Activity model shipped — see DecisionLog AD-14 and IssuesTracker ISSUE-009.

---

### BM-17 — Bare Identifiers in Inline `onclick=`/`oninput=""` Handlers Can Silently Resolve to `document`/`window`, Not Your Own Globals

**Rule:** Never name a page-level global variable or function `children`, `removeChild`, `name`, `open`, `close`, `history`, `top`, `self`, `location`, `status`, or any other real `document`/`window` property — especially one that will ever be referenced as a **bare identifier** inside an inline `onclick=`/`oninput=`/`onchange=""` HTML attribute string.

**Why:** Per the HTML spec, inline event-handler attributes execute inside an implicit `with(document){ with(form){ with(element){ ... } } }` scope. Real built-ins on those objects (`document.children`, `document.removeChild`, `window.name`, `window.history`, etc.) silently shadow a same-named page-level global — the handler runs with no thrown error in the common case, but quietly mutates the DOM instead of your app's actual state.

**Diagnosis method that worked, fast:** before touching the real codebase, write a two-line isolated repro — a bare HTML page with nothing but the suspect global (`let children = [...]`) and one inline handler mutating it — and check what the handler actually mutated. This confirmed the exact mechanism directly in a couple of minutes, instead of guessing at render-cycle or event-binding theories first.

**Symptom if forgotten:** a form field looks like it accepts input (the native browser textbox visibly updates as you type) but the value never persists anywhere the app's own state can see it; a delete/remove button silently does nothing, or throws `Cannot set properties of undefined` for anything past the very first item in a list.

**Where this bit us:** `sports-dashboard.html`'s Settings tab, Add/Edit/Delete Child — see DecisionLog AD-13 and IssuesTracker ISSUE-008. Fixed by renaming the global array `children` → `kids` and the function `removeChild()` → `deleteChild()`.

---

### BM-18 — Settings That Sync to an Unauthenticated Backend Endpoint Must Explicitly Strip Secrets Before Every Write

**Rule:** Anywhere user-editable settings get serialized and sent to a backend endpoint that has no auth (see AD-07 — GAS Web App Access: "Anyone"), explicitly `delete` any secret field from that object immediately before serialization. Don't rely on "we just never put it there" as the only safeguard.

**Why:** BM-15 already covers not committing the API key to git. This is a distinct, newer risk: once Settings started round-tripping through a GAS-backed Sheet (`action=saveSettings` / `action=getSettings`) so a new browser could pull configuration down without manual re-entry, that Sheet became reachable by anyone who has the Web App URL — same as every other endpoint, because AD-07's whole security model is "the URL is the password." A future field added to the settings object (e.g. by copy-pasting a working block that happens to include the key) would leak it silently, with no error and no visible symptom until someone actually has the URL and goes looking.

```javascript
delete settingsObj.ANTHROPIC_API_KEY;
delete settingsObj.apiKey;
```

**Where this applies:** `saveSettings()` in `SportsEmailScanner.gs` — see DecisionLog AD-11. Confirmed this delete is present and correctly placed before the write; recorded here so it's understood as a rule, not just a one-off line of code.

---

## Additions — Session 2026-08-17 (later session)

Three more lessons, same append-only convention as the section above: numbered to continue the existing sequence (next available was BM-19), not physically inserted into the topical sections above.

---

### BM-19 — Always Deploy From the Gmail-Owning Google Account

**Rule:** Confirm you're logged into `tjunker9@gmail.com` before touching Deploy in the Apps Script editor — every time, not just the first time.

**Why:** Apps Script's "Execute as: Me" permanently binds a Web App deployment to whichever Google account was logged in at the moment Deploy was clicked — not to the script project itself. Deploying from a different account (a different Chrome profile, a work/school account, etc.) silently redirects every `GmailApp` call to that account's inbox instead. There's no error at deploy time. It only surfaces later, as `Exception: Gmail operation not allowed`, when a Gmail-touching action (like Refresh Briefing) actually runs — and even manually running the function in the Apps Script editor from the wrong account reproduces the same error with no consent prompt, which makes it look like an auth bug rather than a wrong-account deployment.

**Diagnosis method that worked:** ruled out causes in order rather than guessing — (1) ran the function manually in the editor, got the identical error with no fresh consent screen, which ruled out a stale/revoked token; (2) checked Google Account → linked apps, found full Gmail scope freshly granted, which ruled out missing consent; (3) only then found the actual mismatch — the deploying account wasn't the Gmail-owning account.

**Where this bit us:** see IssuesTracker ISSUE-016 and DecisionLog AD-19.

---

### BM-20 — Deterministic Backstops for Anything the Prompt Asks the Model to Compute

**Rule:** Whenever a Claude prompt asks for a structured field the model has to derive (not just report) — a child-id tag, a computed date — treat the prompt instruction as a first line of defense, not the only one. Add a matching client-side check that discards or corrects results that violate the rule, regardless of what the model returned.

**Why:** This project has now hit this pattern three times independently: ISSUE-010's childIds (a card might get tagged to the wrong kid, or not tagged at all), and this session's ISSUE-011/ISSUE-013 (a past-dated item or a wrong calendar date slipping through despite an explicit prompt rule). Model instruction-following is good, not guaranteed — a rule stated once in a prompt is not the same as a rule enforced in code.

**Where this bit us:** `isPastISO()` in `sports-dashboard.html`, added this session as the backstop for the "no stale content" prompt rule — see DecisionLog AD-16.

---

### BM-21 — Don't Ask the Model to Compute What You Already Know Deterministically

**Rule:** If the browser or server already has the real, correct value for something (e.g. a JS `Date` object for a loaded calendar event), pass that value through directly rather than describing it in prose and asking Claude to recompute it.

**Why:** ISSUE-013 was exactly this mistake — calendar events were sent to the prompt as a display string like `"Mon, Aug 17"` (no year), requiring Claude to infer the actual calendar date itself, which it did unreliably (timeline items landed in an undated "Other" bucket instead of their correct day). The fix wasn't a better prompt — it was removing the need for the model to compute the value at all, by sending the real ISO date through in the `?cal=` payload.

**Where this bit us:** `calEventsParam()` / `calSection` construction in `sports-dashboard.html` and `SportsEmailScanner.gs` — see DecisionLog AD-17.

---

## Session Notes

> Append entries as new lessons are learned. Never delete entries — mark outdated ones as `[Superseded]`.

- **Session 1 (Mar 2026):** File created. BM-01 through BM-15 documented from MVP development.
- **Session — 2026-08-17:** Added BM-16, BM-17, BM-18 from the Child→Activity Settings rebuild and its two bug fixes. Marked BM-08 `[Superseded by AD-10]` — the ICS proxy waterfall it described was replaced by a server-side GAS fetch. Added dated notes to BM-11 and BM-13 confirming both are still open/unapplied as of this session.
- **Session — 2026-08-17 (later session):** Added BM-19 (deployment account discipline, from root-causing ISSUE-016), BM-20 and BM-21 (deterministic backstops / don't make the model compute what you already know, from ISSUE-011 and ISSUE-013).
