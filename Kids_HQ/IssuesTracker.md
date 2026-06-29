# HQ Dashboard — Data Issues & Bug Tracker

**Last Updated:** March 2026  
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

**Expected Behavior:**  
Manual updates should be included in the `generateSummaryWithClaude()` prompt alongside emails and calendar events.

**Proposed Fix:**  
1. Read textarea values in `generate()` JS function
2. Encode as a `?manual=[JSON]` GET param (or append to `?cal=` param)
3. In GAS `doGet()`, parse manual content and add to the prompt as a third section: `=== MANUAL UPDATES ===`

**Notes:** Need to be careful about URL length limits. Could merge manual content into the cal param or send separately.

---

### ISSUE-002 — allorigins.win Proxy Returns Stale Cached ICS Data
**Severity:** 🟡 Medium  
**Status:** Monitoring  
**Filed:** Session 1 (Mar 2026)

**Description:**  
allorigins.win caches responses. ICS feeds fetched via this proxy may be hours old, causing outdated calendar data to show in the dashboard. Recently added or cancelled events may not appear.

**Expected Behavior:**  
Calendar feeds should reflect the current state of the ICS source within a reasonable window (< 1 hour).

**Proposed Fix Options:**  
1. Add a cache-busting query param to the ICS URL before proxying (e.g. `?_t=timestamp`)
2. Move to corsproxy.io as the first proxy (less aggressive caching, though less reliable)
3. Route calendar fetching through GAS instead of browser-side proxies

**Notes:** This is hard to reproduce consistently. Monitor for user-reported stale data.

---

### ISSUE-003 — LeagueApps ICS Feed UTC Offset
**Severity:** 🟡 Medium  
**Status:** Fixed (Session 1)  
**Filed:** Session 1 (Mar 2026)

**Description:**  
LeagueApps and some other feeds used UTC timestamps with `Z` suffix in DTSTART fields. The original ICS parser treated these as local time, causing all events to appear 5-6 hours early (CDT offset).

**Fix Applied:**  
Updated `parseICS()` to detect `Z` suffix and use proper UTC parsing:
```javascript
if (dtRaw.endsWith('Z')) {
  dt = new Date(dtRaw.replace(/(\d{4})(\d{2})(\d{2})T(\d{2})(\d{2})(\d{2})Z/,'$1-$2-$3T$4:$5:$6Z'));
}
```
JavaScript's `Date` then converts to local time automatically.

**Status Notes:** Verified working for CDT. Re-test when clocks change (CST = UTC-6).

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

**Current Regex:**
```javascript
const gcEmbed = url.match(/calendar\.google\.com\/calendar\/(?:embed|htmllink|r)\?.*[?&]src=([^&]+)/i);
```

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

**Expected Behavior:**  
After a successful generate, `historyLoaded` should be reset so the next visit to Email History fetches fresh data.

**Proposed Fix:**  
In `generate()`, after successful completion: `historyLoaded = false;`

---

### ISSUE-007 — Long Email Bodies Truncated at 800 Chars in GAS
**Severity:** 🟢 Low  
**Status:** Open / Won't Fix (Intentional)  
**Filed:** Session 1 (Mar 2026)

**Description:**  
`scanSportsEmails()` truncates email body text to 800 characters. Long emails (detailed practice schedules, tournament brackets, registration forms) may lose important details.

**Current Behavior:**  
```javascript
.substring(0, 800)
```

**Notes:**  
This is intentional to control prompt length and Claude API cost. However, 800 chars may be too short for some content types. Increasing to 1200-1500 chars would increase prompt tokens but likely improve summary quality for complex emails.

**Decision Needed:** Is the truncation threshold acceptable? Any user-reported missed information?

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
- [ ] Email History shows emails from the expected date range
- [ ] Settings tab: test connection shows ✓ Connected
- [ ] All .ics URLs still valid (LeagueApps, TeamSnap, Playmetrics tokens don't expire silently)

---

## Session Notes

> Add notes when issues are opened, updated, or closed.

- **Session 1 (Mar 2026):** Tracker initialized. ISSUE-003 already fixed. ISSUE-001 is the highest priority open item.

