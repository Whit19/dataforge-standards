# HQ Dashboard — Data Issues & Bug Tracker

**Last Updated:** August 2026  
**Purpose:** Track known bugs, data quality issues, edge cases, and open questions that need investigation or fixing.

---

## How to Use This Tracker

**Severity levels:**
- 🔴 **Critical** — Breaks core functionality; fix immediately
- 🟠 **High** — Significant impact on usability; fix soon
- 🟡 **Medium** — Noticeable problem but workaround exists
- 🟢 **Low** — Minor polish / edge case

**Status values:** Open | In Progress | Fixed | Won't Fix | Monitoring

---

## Open Issues

---

### ISSUE-001 — Manual Updates Not Passed to Claude Prompt
**Severity:** 🟠 High  
**Status:** Open  
**Filed:** Session 1 (Mar 2026)

**Description:**  
The Manual Updates tab allows users to paste content from any sports app. However, this content is never included in the Claude prompt when generating a briefing. The textarea values are collected but not sent to GAS.

**Update (2026-08-17):** Still open — reconfirmed by inspecting `generate()` and both prompt builders in the current unified dashboard; no reference to `e-teamsnap` etc. outside the tab's own markup. Additionally, this tab's 7 hardcoded app text areas (TeamSnap/SportsEngine/GameChanger/GroupMe/Playmetrics/League/Carpool Kids) now predate the Child→Activity/sender model from Decision Log AD-12 — they don't necessarily reflect what the user has actually configured as senders anymore. Recommend reworking both gaps together: rebuild this tab to generate one textarea per configured Activity (using its senders/name), then wire the content into the prompt.

**Proposed Fix:**  
1. Rebuild the Manual Updates tab to render one entry per configured Activity, not a hardcoded app list
2. Read those values in the `generate()` JS function
3. Encode as a `?manual=[JSON]` GET param (or fold into the existing `?cal=` param)
4. In GAS `doGet()`, parse manual content and add it to the prompt as a third section

---

### ISSUE-004 — GAS Generate Call Timeout on Slow Networks
**Severity:** 🟡 Medium  
**Status:** Open  
**Filed:** Session 1 (Mar 2026)

**Description:**  
The `action=generate` call can take 30-60 seconds (Gmail scan + Claude API call). On slow mobile connections, the 90-second client timeout occasionally fires, showing an error even when GAS successfully completed the job in the background.

**Expected Behavior:**  
User should see the fresh briefing even if the initial fetch times out, as long as GAS completed successfully.

**Proposed Fix:**  
Implement a polling pattern: on timeout during generate, auto-retry `doGet()` (default — returns stored summary) every 10 seconds for up to 3 attempts. If a fresh `generatedAt` timestamp appears, render it.

**Notes:** Low-priority until user-reported; 90s timeout covers most cases.

---

### ISSUE-005 — Google Calendar Embed URL Conversion Fragile
**Severity:** 🟡 Medium  
**Status:** Open  
**Filed:** Session 1 (Mar 2026)

**Description:**  
The `normalizeIcsUrl()` function attempts to convert Google Calendar HTML embed URLs to `.ics` format. The regex pattern handles common URL shapes but may miss newer Google Calendar URL formats.

**Known Gap:**  
Google Calendar's "Get shareable link" URL format (`/calendar/r/...`) uses different parameters in some cases. User may paste a URL that looks like it should work but silently fails.

**Proposed Fix:**  
Add better error messaging when a Google Calendar URL fails to convert. Show user what the expected .ics format looks like.

---

### ISSUE-006 — Email History Not Refreshed After Generate
**Severity:** 🟢 Low  
**Status:** Open  
**Filed:** Session 1 (Mar 2026)

**Description:**  
If the user clicks "Refresh Briefing" (which rescans Gmail), then navigates to the Email History tab, they still see the old history. The `historyLoaded` flag prevents a re-fetch.

**Update (2026-08-17):** Reconfirmed still present — `historyLoaded=false` is currently only reset inside `setCategory()` (switching the Sports/School tab), not after a successful `generate()`.

**Proposed Fix:**  
In `generate()`, after successful completion: `historyLoaded = false;`

---

### ISSUE-007 — Long Email Bodies Truncated at 800 Chars in GAS
**Severity:** 🟢 Low  
**Status:** Open / Won't Fix (Intentional)  
**Filed:** Session 1 (Mar 2026)

**Description:**  
`cleanEmailBody()` truncates email body text to 800 characters. Long emails (detailed practice schedules, tournament brackets, registration forms) may lose important details.

**Notes:**  
This is intentional to control prompt length and Claude API cost. However, 800 chars may be too short for some content types.

**Decision Needed:** Is the truncation threshold acceptable? Any user-reported missed information?

---

### ISSUE-010 — Briefing Child Tagging Depends on the AI Following Instructions Correctly
**Severity:** 🟡 Medium  
**Status:** Monitoring  
**Filed:** 2026-08-17

**Description:**  
As of the fix landing under Decision Log PD-04, Claude is asked to tag each briefing card/timeline entry/action with a `childIds` array, using an id legend fed into the prompt. This is model output, not deterministic code — there's no hard guarantee Claude always tags correctly or consistently.

**Mitigation already in place:**  
Untagged items (`childIds` missing or empty) are treated as "general" and shown under every filter, including a specific child — so a tagging miss makes an item show up too often, never disappear. Verified with an automated test covering: a card tagged for one child, one for another, one explicitly general, and one from an old cached briefing with no `childIds` field at all (pre-dates this change) — all four behaved as expected.

**What to watch for:** if a user reports a card clearly appears under the wrong child (not just "too often"), that's a real tagging-quality issue worth revisiting (e.g. tightening the prompt's rules section), not a code bug.

---

## Closed / Fixed Issues

---

### ISSUE-002 — allorigins.win Proxy Returns Stale Cached ICS Data
**Severity:** Was 🟡 Medium  
**Status:** Fixed (2026-08-17)  
**Filed:** Session 1 (Mar 2026)

**Description:**  
allorigins.win caches responses. ICS feeds fetched via this proxy could be hours old, and separately the whole proxy waterfall (allorigins → corsproxy.io → codetabs) became unreliable in production — direct testing showed allorigins timing out and corsproxy.io no longer proxying the old query format at all.

**Resolution:**  
Calendar `.ics` fetching moved server-side through GAS (`UrlFetchApp`, no CORS restriction, no dependency on third-party proxy uptime/API stability). See Decision Log AD-10, which supersedes AD-05.

---

### ISSUE-003 — LeagueApps ICS Feed UTC Offset
**Severity:** Was 🟡 Medium  
**Status:** Fixed (Session 1)  
**Filed:** Session 1 (Mar 2026)

**Description:**  
LeagueApps and some other feeds used UTC timestamps with `Z` suffix in DTSTART fields. The original ICS parser treated these as local time, causing all events to appear 5-6 hours early (CDT offset).

**Fix Applied:**  
Updated `parseICS()` to detect `Z` suffix and use proper UTC parsing. JavaScript's `Date` then converts to local time automatically.

**Status Notes:** Verified working for CDT. Re-test when clocks change (CST = UTC-6).

---

### ISSUE-008 — `children`/`removeChild` Globals Silently Shadowed by `document` Built-ins
**Severity:** Was 🔴 Critical  
**Status:** Fixed (2026-08-17)  
**Filed:** 2026-08-17

**Description:**  
Typing a child's name in Settings appeared to work but never persisted; the delete (✕) button didn't delete. Root cause was NOT a rendering bug — it was a JS scoping collision: inline `oninput`/`onclick` handlers referencing the bare identifiers `children` and `removeChild` were silently resolving to `document.children` and `document.removeChild` (real DOM built-ins) instead of the page's own global array/function, per the implicit `with(document){...}` scope inline HTML event handlers execute in.

**Diagnosis method:** confirmed with an isolated two-line Playwright repro (a page with nothing but `let children=[{name:''}]` and an inline `onclick` mutating `children[0].name`) before writing any fix, per this project's standing debugging approach of finding the actual cause first.

**Fix Applied:**  
Renamed the global array to `kids` and the delete function to `deleteChild()` throughout `sports-dashboard.html`. Audited every other inline handler in the file for the same class of collision (none further found). Verified with an automated test: add 3 children, confirm they persist in state and render in both the child pill switcher and each activity's chip picker, then delete one and confirm it's actually gone — zero console errors.

**See:** Decision Log AD-13 for the standing rule this leaves behind.

---

### ISSUE-009 — Rolling Email History Never Re-Tagged Existing Rows After a Settings Reorg
**Severity:** Was 🟠 High  
**Status:** Fixed (2026-08-17)  
**Filed:** 2026-08-17

**Description:**  
Per-child email filtering worked for some kids and not others. Root cause: `updateHistorySheet()`'s merge logic kept an email's originally-assigned `ActivityId` forever once it was first written to the sheet — only genuinely new `date|subject` keys got added, so re-tagging an email after linking its sender to a real child in Settings never reached emails already sitting in history.

**Fix Applied:**  
Merge now keys by `date|subject` and lets the current scan's row win on collision, since `scanEmails()` already fully re-derives correct tagging for the whole `daysBack` window on every run.

**See:** Decision Log AD-14.

---

## Closed / Won't Fix Issues

---

### ISSUE-C01 — POST from Browser to GAS CORS-Blocked
**Severity:** Was 🔴 Critical  
**Status:** Won't Fix (by design — see AD-02)  
**Filed:** Session 1 (Mar 2026)

**Description:**  
Browser POST requests to GAS Web App endpoints fail CORS preflight in strict browsers (Chrome, Safari). Calendar events originally sent via POST never arrived.

**Resolution:**  
Switched to GET with `?cal=` parameter encoding. POST endpoint (`doPost()`) remains in GAS for legacy compatibility but is no longer used by the dashboard. See Decision AD-02.

---

## Monitoring Checklist

Run these checks periodically or after any GAS / dashboard update:

- [ ] Load Calendars → all teams appear with correct event counts
- [ ] Events show at correct local time (test with a known event)
- [ ] Refresh Briefing completes within 90 seconds
- [ ] Summary contains cards from multiple senders (not just one app)
- [ ] Child filter pill actually changes what's shown on Calendars, Email History, *and* Summary
- [ ] Add / edit / delete a Child in Settings and confirm it sticks after a page reload
- [ ] Email History shows emails from the expected date range, correctly attributed to the right child
- [ ] Settings tab: test connection shows ✓ Connected
- [ ] "Load Settings from Cloud" round-trips correctly in a fresh/incognito browser
- [ ] All .ics URLs still valid (LeagueApps, TeamSnap, Playmetrics tokens don't expire silently)

---

## Session Notes

> Add notes when issues are opened, updated, or closed.

- **Session 1 (Mar 2026):** Tracker initialized. ISSUE-003 already fixed. ISSUE-001 is the highest priority open item.
- **Session — 2026-08-17:** ISSUE-002 fixed (GAS-side ICS fetch, see AD-10). Two new critical/high bugs found, root-caused, fixed, and verified with automated tests: ISSUE-008 (child add/delete silently broken by a DOM-shadowing collision) and ISSUE-009 (email history never re-tagged after Settings reorg). Added ISSUE-010 to track the new AI-dependent child-tagging feature as a monitoring item, not a guaranteed-correct one. ISSUE-001 remains the top open item and now has an added dimension (its hardcoded app list is stale against the Activity/sender model).
