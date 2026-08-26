# HQ Dashboard — Best Methods

**Last Updated:** 2026-08-26  
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

### BM-22 — Secrets Live in PropertiesService, Never as a Hardcoded Const, Regardless of Repo Visibility

**Rule:** Any script-level secret (`ANTHROPIC_API_KEY`, `GROUPME_ACCESS_TOKEN`, or any future one) is read via `PropertiesService.getScriptProperties().getProperty(...)` at runtime, set manually through the Apps Script editor's Project Settings → Script Properties UI — never as a literal string assigned to a const in the `.gs` file itself.

**Why:** BM-15's original safeguard ("the GAS script isn't stored in GitHub") turned out to be a false assumption in practice, not a durable boundary — see the correction above and IssuesTracker ISSUE-017. `PropertiesService` removes the secret from any file that could ever be committed at all, which doesn't depend on remembering the repo is private, remembering to `.gitignore` something, or any other manual discipline that can lapse.

**Where this bit us:** `SportsEmailScanner.gs` — see DecisionLog AD-21.

---

### BM-23 — A Private GitHub Repo Is Not a Durable Secret Boundary

**Rule:** Never treat "the repo is private" as sufficient protection for a secret. Rotate immediately if one leaks into a commit, regardless of the repo's visibility setting, and don't rely on visibility as the primary safeguard going forward.

**Why:** A private repo's protection depends on things that can silently change or be bypassed — a collaborator added later, an accidental visibility toggle, a compromised GitHub account, a forked/cloned copy nobody remembers exists. None of those require the repo to ever go technically "public" for the secret to be exposed. Confirmed as the actual failure mode here: `SportsEmailScanner.gs` was tracked in a private repo the whole time — privacy alone did not prevent the key from being a real leaked credential once committed.

**Where this bit us:** IssuesTracker ISSUE-017; referenced directly in DecisionLog AD-21's rationale.

---

### BM-24 — Recommended, Not Yet Implemented: a `gitleaks` Pre-Commit Hook

**Rule (recommendation, not yet actioned):** Add a `gitleaks` pre-commit hook (or equivalent secret-scanning tool) to this repo so a credential-shaped string is caught *before* a commit is made, not after a push.

**Why:** ISSUE-017's leak wasn't caught until after it was already pushed to the remote — rotation and history-scrubbing are cleanup, not prevention. A pre-commit scanner would have caught the `ANTHROPIC_API_KEY` const before it ever left the local machine.

**Status:** Not yet implemented — logged here as a known gap, not a completed safeguard. Revisit alongside any future GAS/dashboard session that touches secrets handling.

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

**Where this bit us:** `calEventsParam()` / `calSection` construction in `sports-dashboard.html` and `SportsEmailScanner.gs` — see DecisionLog AD-17. Note as of 2026-08-19: `calEventsParam()` itself was removed entirely when the calendar data source changed (AD-23) — this entry's lesson still stands, it just no longer applies to a `?cal=` payload specifically since that payload no longer exists for the briefing.

---

## Additions — Session 2026-08-19

Three new lessons: BM-25 extends the model-output-reliability pattern (BM-20) to a second tagged field; BM-26 and BM-27 are new lessons from replacing the `.ics` calendar pipeline with a direct Google Calendar scan (DecisionLog AD-23). Same append-only convention as prior session sections.

---

### BM-25 — The Model-Output-Reliability Pattern Extends to Every New Tagged Field, Not Just the First One

**Rule:** When a second (or third) structured field gets added to what Claude is asked to tag on its own output — following the precedent BM-20 established for `childIds` and `dateISO` — give it the exact same treatment: a legend of valid values in the prompt, an explicit "don't guess, empty/blank means unclear" instruction, and a graceful client-side degradation rule (untagged shows under every filter, never disappears) rather than assuming a new field will just work because the pattern worked before.

**Why:** `activityId` (DecisionLog PD-06) is the third field to get this treatment. Each one is a separate opportunity for the model to under-tag, over-tag, or misapply — treating "we already solved this once" as covering the new field too would be assuming reliability that was never actually re-verified for it.

**Where this applies:** `briefingItemMatches()` in `sports-dashboard.html`, extended to check `activityId` alongside `childIds` — see IssuesTracker ISSUE-020 for the corresponding monitoring entry.

---

### BM-26 — Match Strategy Should Follow the Data Source's Actual Structure, Not Be Forced Uniform Across It

**Rule:** When a single system exposes some data through a dedicated, purpose-scoped container (a named calendar that only ever contains one team's events) and other data through a shared, mixed container (a personal calendar with everything in it), don't apply the same matching heuristic to both. Use the container's own identity as the primary signal wherever a dedicated container exists — no content-matching needed there — and reserve content-based matching (title, description, etc.) for the mixed container where no better signal is available.

**Why:** An earlier draft of `scanGoogleCalendar()` used title-only keyword matching everywhere, which would have missed real events on dedicated team calendars whose titles don't happen to mention the team, AND missed events on the personal calendar where the activity name only appears in the description (a ForeTees tee-time confirmation), not the title. Two different pass strategies, matched to two structurally different data sources, caught both cases; one uniform strategy would have caught neither reliably.

**Where this applies:** `scanGoogleCalendar()`'s two-pass design in `SportsEmailScanner.gs` — see DecisionLog AD-23.

---

### BM-27 — A New GAS Built-In Service Needs a Manual One-Time Authorization Run Before the First Deploy

**Rule:** The first time a GAS script calls a built-in service it's never used before (`CalendarApp`, `DriveApp` beyond the existing `getFilesByName` fallback, etc.), run any function containing that call once manually from the Apps Script editor — not via the Web App — and approve the resulting permission prompt, *before* deploying a new version.

**Why:** A Web App request can't interactively prompt an anonymous caller for OAuth consent. If the script calls a not-yet-authorized service from a live `doGet`, it fails silently in a way that's easy to miss: `scanGoogleCalendar()` wraps its `CalendarApp` calls in try/catch and logs a warning rather than throwing, per this project's own established pattern (BM-01's `muteHttpExceptions` philosophy applied one level up) — so `generate` still returns a briefing, just with zero calendar events, no error visible to the person using the dashboard.

**Where this applies:** `CalendarApp.getDefaultCalendar()` / `getAllCalendars()`, first introduced this session — see DecisionLog AD-23 and SessionStarter.md's First-Time Setup step 5.

---

## Additions — Session 2026-08-21

Three new lessons: BM-28 is a calendar-matching gotcha found while
completing this session's testing checklist; BM-29 and BM-30 are debugging
techniques discovered while working around an inaccessible Apps Script
Cloud Logs UI. Same append-only convention as prior session sections.

---

### BM-28 — A Personal Calendar Rename Never Changes What `CalendarApp` Reads for a Calendar You Don't Own

**Rule:** Renaming a subscribed (not owned) calendar via Google Calendar's
own Settings UI only changes your personal display of that calendar — it
does not rename the calendar's underlying identity. `CalendarApp.getAllCalendars()`
always returns the name the calendar's actual owner assigned it, regardless
of how you've relabeled it in your own view.

**Why:** This session confirmed directly: the "Bavarian Soccer" Activity's
named-calendar match (AD-23 Pass 1) kept failing even after renaming the
calendar to "U17/U18 Bavarian Soccer" in Google Calendar's UI. A debug
dump of `CalendarApp.getAllCalendars()`'s actual returned names showed the
calendar was still reporting as "U17/U18 Girls Blue" — its real name, set
by the Playmetrics-synced team calendar's owner, completely unaffected by
the personal rename. This is the same underlying mechanism already
described at a symptom level in IssuesTracker ISSUE-020 (the WNS Lax
naming-mismatch case) — this entry captures the actual cause, not just the
symptom.

**Where this bit us:** `scanGoogleCalendar()`'s Pass 1 matching in
`SportsEmailScanner.gs` — see IssuesTracker ISSUE-020's 2026-08-21
addendum and DecisionLog AD-25 (the `calMatchName` field this lesson
motivated).

**Takeaway for future matching problems:** if a calendar match fails
despite a calendar's name visibly matching an Activity in the Google
Calendar UI, check the ACTUAL name via a debug dump before assuming the
matching logic itself is broken — the UI name and the API-visible name can
silently diverge for any calendar you don't own.

---

### BM-29 — When Apps Script Cloud Logs Are Inaccessible, Use the Browser's Network Tab as a Fallback Diagnostic Channel

**Rule:** If the Executions tab's Cloud Logs panel won't expand or load
(greyed out, unresponsive, "No logs are available" that never resolves),
add a small temporary debug object to the JSON response `doGet` already
returns for the request being diagnosed, then inspect it directly via the
browser's DevTools → Network tab on the dashboard's own request — no Apps
Script UI involved at all.

**Why:** This session hit a genuinely broken Cloud Logs panel (three-dot
menu's "Cloud logs" option greyed out, row expansion showing "No logs are
available for this execution" indefinitely) with no clear cause. Rather
than keep fighting the GAS UI, a temporary `_debugCalInfo` field added to
the `action=generate` response — read via DevTools Network tab → find the
request → Preview tab — gave full visibility in minutes.

**Important caveat confirmed this session:** GAS Web App responses are
often a 302 redirect to a temporary `googleusercontent.com` URL. The
`exec?...` request itself may show "Failed to load response data... this
request was redirected" — the actual JSON body is on the FOLLOWING
`echo?user_content_key=...` request in the Network list, not the `exec`
request itself.

**Where this applies:** general debugging technique, not tied to one file
— used this session to diagnose the Bavarian Soccer calendar-matching
issue (BM-28). Any temporary debug field added this way must be removed
in a dedicated follow-up prompt once diagnosis is complete — see
CC_01_RemoveTempDebug.md from this session as the pattern to follow.

---

### BM-30 — Check Execution Duration Against a Known Benchmark Before Trusting "No Logs Available"

**Rule:** Before concluding a Web App execution's logs are missing or
broken, check its Duration column first. A `doGet` finishing in roughly
1–2 seconds is almost certainly the cheap default-read path
(`?cat=sports`, no `action` param — just returns the already-stored
Summary sheet) and genuinely has little to log. Only a duration in the
~30–90+ second range reflects a real `action=generate` run (Gmail scan +
calendar scan + Claude API call, per BestMethods BM-09's documented
timeout budget) and is worth chasing logs for.

**Why:** This session repeatedly investigated "why are there no logs" on
`doGet` executions that turned out to be 1.2–2.7 second default-read
requests — not because logging was broken, but because those particular
requests never triggered a real scan or Claude call in the first place. A
73.517-second row, once identified by duration alone, was the one that
actually mattered and matched independently-confirmed real generate
behavior.

**Where this applies:** general Executions-tab triage — check Duration
before assuming Cloud Logs are broken or missing.

---

## Additions — Session 2026-08-26

Three new lessons from this session's UI/PWA polish pass on Summary and
Calendars. Numbered to continue the existing sequence (next available was
BM-31). Same append-only convention as prior session sections.

---

### BM-31 — Google Sheets Silently Coerces Date/Time-Shaped Strings Into Real Date Objects on Write

**Rule:** Writing a plain string like `"2026-08-26"` or `"9:00 AM"` to a Sheets cell (e.g. via `setValue`) gets auto-converted to an actual `Date` value, the same as if it had been typed into the sheet by hand. A later `getValues()`/`getValue()` read then returns a JS `Date` object, not the original string — and `JSON.stringify`-ing that produces a full ISO timestamp (`"2026-08-26T00:00:00.000Z"`), not the expected plain-string shape. Any endpoint reading date/time-formatted cells back out of a Sheet should explicitly check the returned type and reformat `Date` objects to the exact string shape the frontend expects, rather than assuming `getValues()` returns what was originally written.

**Why:** This caused IssuesTracker ISSUE-025 — a `doGet` endpoint fetch that "succeeded" (events loaded) but silently failed a client-side string-equality date filter, with no error anywhere in the chain.

**Where this bit us:** the new `action=calendarEvents` endpoint in `SportsEmailScanner.gs` — see DecisionLog AD-29.

---

### BM-32 — Flex `order` vs. `margin-left:auto` Can Fight Each Other in Ways That Look Like a Margin Bug

**Rule:** When a flex-row element with `margin-left:auto` appears to be misplacing sibling elements rather than just itself, check its position in flex order before adjusting margins.

**Why:** IssuesTracker ISSUE-027 — a badge element with `margin-left:auto` but no explicit flex `order` was dragging its entire parent row rightward, not just itself, because it was first in DOM/flex order. Setting `order:1` on the badge (moving it to the end of the flex flow) fixed it without touching the margin at all.

**Where this bit us:** the Calendars week view's "Today" row in `sports-dashboard.html`.

---

### BM-33 — GitHub Actions Incidents Can Leave a Workflow Run Stuck "Queued" With Cancel Also Failing

**Rule:** If a GitHub Pages deploy workflow gets stuck "Queued" and manual cancellation also fails, check githubstatus.com before assuming a local/repo-side problem. No local fix exists for this — the workaround is a trivial recommit (e.g. a one-line comment change) once GitHub's status page shows the incident resolved, which triggers a fresh workflow run rather than waiting on or fighting the stuck one.

**Why:** During a live GitHub Actions incident (Aug 26, 2026, ~15:11–18:01 UTC per githubstatus.com), a `pages-build-deployment` run got stuck in "Queued" and manual cancellation failed too — both the queuing and cancellation paths were affected by the same incident.

**Where this applies:** general GitHub Actions/Pages deployment triage, not tied to one file.

---

## Session Notes

> Append entries as new lessons are learned. Never delete entries — mark outdated ones as `[Superseded]`.

- **Session 1 (Mar 2026):** File created. BM-01 through BM-15 documented from MVP development.
- **Session — 2026-08-17:** Added BM-16, BM-17, BM-18 from the Child→Activity Settings rebuild and its two bug fixes. Marked BM-08 `[Superseded by AD-10]` — the ICS proxy waterfall it described was replaced by a server-side GAS fetch. Added dated notes to BM-11 and BM-13 confirming both are still open/unapplied as of this session.
- **Session — 2026-08-17 (later session):** Added BM-19 (deployment account discipline, from root-causing ISSUE-016), BM-20 and BM-21 (deterministic backstops / don't make the model compute what you already know, from ISSUE-011 and ISSUE-013).
- **Session — 2026-08-18 (backfilled 2026-08-19 — this entry, and BM-22/23/24 themselves, were referenced by DecisionLog AD-21 and IssuesTracker ISSUE-017 since this date but never actually written into this file until now):** Added BM-22, BM-23, BM-24 from the leaked-API-key incident (rotate → Script Properties migration → git history scrub → prevention recommendation).
- **Session — 2026-08-19:** Added BM-25 (model-output-reliability pattern extended to `activityId`), BM-26 (matching strategy should follow data-source structure, not be forced uniform), and BM-27 (new GAS service needs a manual one-time authorization run before deploy) — all three from replacing the `.ics` calendar pipeline with a direct Google Calendar scan (DecisionLog AD-23).
- **Session — 2026-08-21:** Added BM-28 (personal calendar renames don't
  change what CalendarApp reads for calendars you don't own — the real
  root cause behind the Bavarian Soccer/WNS Lax matching failures), BM-29
  (Network tab as a fallback diagnostic channel when Cloud Logs UI is
  inaccessible, including the exec→echo redirect gotcha), and BM-30
  (check execution duration against known benchmarks before assuming logs
  are broken) — all three from this session's calendar-scan testing and
  debugging work.
- **Session — 2026-08-26:** Added BM-31 (Google Sheets silently coerces
  date/time-shaped strings into real Date objects on write — the root
  cause behind ISSUE-025), BM-32 (flex `order` vs. `margin-left:auto` can
  fight each other in ways that look like a margin bug), and BM-33
  (GitHub Actions incidents can leave a workflow stuck "Queued" with
  cancel also failing) — from this session's Summary/Calendars UI/PWA
  polish pass.
