# HQ Dashboard — Data Issues & Bug Tracker

**Last Updated:** 2026-08-26  
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

**Update (2026-08-17, later session):** Confirmed still relevant — Tom flagged Refresh Briefing feeling slow. A completed refresh was observed finishing well inside the 90s window (not an active failure, just wait-time UX). Deliberately deferred to its own future session rather than folded into an unrelated fix.

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

**Update (2026-08-17, later session):** Still open — NOT incidentally fixed by that session's Category→Child→Activity filter rework. That work changed what Email History filters by, not when it refetches; `historyLoaded=false` is still only reset inside `setCategory()`, never after a successful `generate()`. Flagging explicitly so this isn't wrongly assumed resolved.

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

### ISSUE-020 — Google Calendar Matching Depends on Calendar/Event Naming Conventions
**Severity:** 🟡 Medium  
**Status:** Monitoring  
**Filed:** 2026-08-19

**Description:**  
As of DecisionLog AD-23, the briefing's calendar events come from a two-pass heuristic match (calendar name, then event title, then event description) rather than an explicit per-event link the way `.ics` calendars were tied to an Activity via a dropdown. There's no hard guarantee every real event gets matched, or matched to the correct Activity.

**Known limitation already identified:** calendar-name matching (Pass 1) is a loose bidirectional substring check — a calendar named "WNS Girls Lax Schedule 2…" will NOT match an Activity named "WNS Lax," since neither string is a substring of the other, even though they're clearly the same team to a person.

**Mitigation already in place:** an event matching no configured Activity is dropped rather than shown unassigned — prevents noise, at the cost of possibly under-including rather than over-including.

**What to watch for:** check the GAS Executions log line (`Google Calendar: matched X event(s)...`) after a Refresh Briefing — if a calendar or event you expected to see is missing, the fix is renaming the Activity in Settings to more closely match the calendar's or event's actual wording, not new code. See SessionStarter.md's testing checklist for this session's specific verification steps.

**Addendum (2026-08-21):** Root cause for a real instance of this
limitation confirmed directly. The "Bavarian Soccer" Activity wasn't
matching its named "U17/U18 Girls Blue" calendar even though Tom had
renamed the calendar in his own Google Calendar UI to "U17/U18 Bavarian
Soccer" — because that rename is only a personal display override for a
calendar he doesn't own; `CalendarApp` still reads the calendar's real
underlying name regardless of how it's relabeled in his own view. Same
root cause applies to the existing WNS Lax example. Interim fix applied:
renamed the Activities themselves to match the calendars' real names.
Durable fix designed: a separate `calMatchName` field (Decision Log AD-25)
so the calendar-matching term can differ from the display name — prompted,
not yet deployed.

---

### ISSUE-021 — Relative Date Language Resolved Against Today Instead of the Source Message's Own Date
**Severity:** 🟡 Medium
**Status:** In Progress — fix prompted, not yet deployed
**Filed:** 2026-08-21

**Description:**
A briefing card ("Brooke — Parent/Player Meeting Tuesday") showed the
wrong date in one run (Aug 25 instead of the correct Aug 18) and garbled,
conflated text in a later run ("rescheduled from tomorrow," referencing an
unrelated message). Root cause, confirmed by reading the actual source
GroupMe thread: an Aug 10 message stated the date explicitly ("Tuesday,
August 18th"), but an Aug 12 follow-up just said "next Tuesday" with no
explicit date — and Claude resolved that relative language against
today's scan date instead of the Aug 12 message's own send date. This is
distinct from AD-16, which only governs dropping already-passed content,
not resolving ambiguous/relative date language.

**Proposed Fix:**
Add an explicit prompt rule to both `buildSportsPrompt` and
`buildSchoolPrompt` instructing Claude to resolve relative date language
against the source message's own date, and to prefer an earlier message's
explicit date over recomputing one from later relative language in the
same thread. See CC_02_RelativeDateFix.md.

---

### ISSUE-022 — Stale `scanSportsEmails` Trigger Failing Daily
**Severity:** 🟢 Low
**Status:** Fixed (2026-08-21) — confirmed, Tom deleted the trigger directly
**Filed:** 2026-08-21

**Description:**
A Time-Driven trigger for a function called `scanSportsEmails` was firing
daily around 6:20 AM and failing every time with "Script function not
found: scanSportsEmails" — a leftover from before the Sports/School
unification (PD-03); no such standalone function exists in the current
codebase (email scanning for both categories happens inside
`runDailyBriefing()` → `scanEmails()`).

**Resolution:**
Deleted directly in the Apps Script Triggers UI — no code change needed.
Confirmed by Tom.

---

### ISSUE-029 — Activity Color Is Not Currently Editable From Settings
**Severity:** 🟢 Low  
**Status:** Open — Deferred to the Settings redesign session  
**Filed:** 2026-08-26

**Description:**  
Each Activity's `color` field (used for the Calendars tab's per-event color bars, auto-assigned from a rotating palette at Activity creation) has no color-picker UI on the Activity card in Settings — only Family Members and individual senders/GroupMe entries currently have one.

**Notes:** Deferred to the upcoming Settings redesign session rather than fixed as a one-off, since Settings is getting a broader UI pass anyway.

---

### ISSUE-030 — `fetchIcsServerSide()`/`action=ics` Backend Handler and the `cals` Config Field Are Dead Code
**Severity:** 🟢 Low  
**Status:** Open  
**Filed:** 2026-08-26

**Description:**  
Left in place intentionally during the Calendars data-source migration (AD-29) to reduce rollout risk. Nothing in the current frontend calls `action=ics` or reads/writes the `cals` Settings field anymore.

**Proposed Fix:**  
Safe to remove in a future cleanup pass once the new `action=calendarEvents` path has been confirmed stable for a while.

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

### ISSUE-011 — Briefing Surfaced Stale/Past-Dated Content From Historical Emails
**Severity:** Was 🟠 High  
**Status:** Fixed (2026-08-17)  
**Filed:** 2026-08-17

**Description:**  
Because the app intentionally feeds Claude the full rolling email history for context, the prompt never instructed Claude to check whether an email's referenced event/deadline had already passed relative to "today" before surfacing it. Observed directly: a briefing generated Monday, Aug 17 included "Sign up Whit for Irish Fest volunteer shift on Saturday Aug 15" as a live action item — two days after the fact.

**Root cause (confirmed, not guessed):** the prompt told Claude what today's date was but had no rule about dropping already-passed items, and the full-history design (intentional, see BestMethods BM-14) means old emails describing since-passed events are always in context.

**Fix Applied:**  
- Both prompt builders now receive `todayISO` and an explicit "no stale content" rule: never surface a card/timeline/action item whose date has already fully passed relative to today, with one exception (a past event whose email still carries forward-relevant info, e.g. a confirmation).
- Timeline and action items now carry a `dateISO` field.
- Added a deterministic client-side filter (`isPastISO()`) that drops any item with a past `dateISO` regardless of what Claude returned — the same don't-fully-trust-the-model pattern established by ISSUE-010.

**See:** Decision Log AD-16.

---

### ISSUE-012 — 7-Day Schedule Was a Flat, Undifferentiated List
**Severity:** Was 🟢 Low (UX)  
**Status:** Fixed (2026-08-17)  
**Filed:** 2026-08-17

**Description:**  
Timeline entries rendered as one flat list with no day grouping — not clear at a glance which day something fell on, or whether a day had nothing scheduled at all.

**Fix Applied:**  
Rewrote as a day-grouped view: one section per day (today + next 6), each showing its items or an explicit "Nothing scheduled" placeholder, so all 7 days are always visible.

---

### ISSUE-013 — Calendar-Derived Timeline Items All Grouped Under "Other" Instead of the Correct Day
**Severity:** Was 🟠 High  
**Status:** Fixed (2026-08-17)  
**Filed:** 2026-08-17 (introduced by the ISSUE-012 fix, caught same session)

**Description:**  
After the day-grouped 7-Day Schedule shipped (buckets items by `dateISO`), calendar-derived events — recurring soccer practices, a golf tee time, a lacrosse tryout — all landed in the undated "Other" bucket instead of their correct day.

**Root cause (confirmed by tracing the data flow, not guessed):** the calendar event payload sent from the dashboard to GAS only ever included a display string like `"Mon, Aug 17"` — no year, no computed ISO date. Claude was left to infer the real calendar date from that string alone in the prompt, and wasn't reliably producing (or including) a matching `dateISO`.

**Fix Applied:**  
- The browser already has a real JS `Date` object for every loaded calendar event; that ISO date is now sent through in the `?cal=` payload (`di` field).
- Stored in a new `DateISO` column on each category's CalendarEvents sheet.
- Surfaced explicitly in the prompt's CALENDAR EVENTS section as `{dateISO: YYYY-MM-DD}` next to each event.
- Prompt now instructs Claude to copy that value directly for calendar-sourced items instead of computing it, and to only compute `dateISO` by hand for email-only events (with a self-check that the stated weekday matches).

**See:** Decision Log AD-17.

---

### ISSUE-014 — Adding a To-Do From an Action Item Didn't Appear Until Page Reload
**Severity:** Was 🟡 Medium  
**Status:** Fixed (2026-08-17)  
**Filed:** 2026-08-17 (introduced by the same session's To-Do feature, caught next follow-up)

**Description:**  
The "+ To-Do" button on Action Items saved the new item to `localStorage` correctly but never re-rendered the To-Do tab's DOM, so the item was invisible until a full page reload.

**Fix Applied:**  
`addTodoFromAction()` now calls `renderTodos()` immediately after saving.

---

### ISSUE-015 — Phantom Vertical Scrollbar on Tab Nav
**Severity:** Was 🟢 Low  
**Status:** Fixed (2026-08-17) — pending visual confirmation  
**Filed:** 2026-08-17

**Description:**  
The Summary/Calendars/Email History/etc. tab row displayed a faint vertical scrollbar that didn't belong to a normal tab bar.

**Root cause (diagnosed via CSS spec behavior, not guessed):** `.tab-nav` set `overflow-x:auto` without setting `overflow-y`. Per the CSS spec, when one overflow axis is set to a non-`visible` value and the other is left as `visible`, the browser computes the unset axis as `auto` too — so any sub-pixel vertical overflow (button padding/line-height) triggered a phantom scrollbar.

**Fix Applied:**  
Added `overflow-y:hidden` explicitly.

**Status caveat:** flagged as "likely, not certain" when shipped — a genuine CSS quirk that matches the symptom exactly, but not yet explicitly confirmed fixed in Tom's browser. Confirm next session.

---

### ISSUE-016 — "Gmail Operation Not Allowed" on Refresh Briefing (Process Issue, Not a Code Bug)
**Severity:** Was 🔴 Critical (blocked the core Refresh Briefing feature entirely)  
**Status:** Fixed (2026-08-17)  
**Filed:** 2026-08-17

**Description:**  
Refresh Briefing began failing with `Exception: Gmail operation not allowed`, while loading the dashboard (which only reads the last stored summary) continued to work fine — the tell that only the live `GmailApp.search()` call was affected.

**Root cause (isolated by elimination, not guessed):**
1. Running the function manually in the Apps Script editor reproduced the identical error with no fresh consent prompt — ruled out a stale/revoked OAuth token.
2. Checking Google Account → linked apps showed full Gmail scope freshly granted (35 minutes old) and the error still persisted — ruled out missing/incomplete user consent.
3. Actual cause: the Web App had most recently been redeployed from a different Google account (`tjunker@dataforge`) than the one whose Gmail inbox the script is meant to scan (`tjunker9@gmail.com`). Apps Script's "Execute as: Me" permanently binds a deployment to whichever account was logged in at the moment Deploy was clicked — so every Gmail operation was silently pointed at the wrong account's inbox.

**Fix Applied:**  
Redeployed the Web App from the `tjunker9@gmail.com` account. No code change was needed or made.

**Notes:**  
This is a standing operational trap, not a one-time fix — deploying from any other account will reproduce this exact failure with no error at deploy time, only later when Refresh Briefing is actually clicked. See Decision Log AD-19 and BestMethods BM-19.

---

### ISSUE-017 — Anthropic API Key Hardcoded in SportsEmailScanner.gs and Pushed to Remote
**Severity:** Was 🔴 Critical  
**Status:** Fixed (2026-08-18)  
**Filed:** 2026-08-18

**Description:**  
A live Anthropic API key was committed as a plaintext const (`ANTHROPIC_API_KEY`) in `SportsEmailScanner.gs` and pushed to `origin/master` (commit `b93fa20`) in the project's private GitHub repo — the same repo that also hosts the GitHub Pages-deployed dashboard. Root cause: BestMethods BM-15 had assumed the GAS script wasn't git-tracked at all, so "the key lives safely in Apps Script cloud" no longer held once the file was actually being committed alongside `sports-dashboard.html`.

**Resolution:**  
1. Rotated the key immediately in the Anthropic Console (old key revoked, new key issued) — done regardless of the repo's actual public/private visibility, since a private repo isn't a durable secret boundary (see BestMethods BM-23)
2. Migrated both `ANTHROPIC_API_KEY` and the newly-added `GROUPME_ACCESS_TOKEN` from hardcoded consts to Apps Script's Script Properties (see Decision Log AD-21) — closes the root cause, not just this one instance
3. Scrubbed the leaked commit from git history via `git-filter-repo` + force-push; verified `b93fa20` is unreachable from any local or remote ref, and `origin/master`'s current copy of the file contains no key string
4. Recommended, not yet actioned: a `gitleaks` pre-commit hook to catch this class of mistake before a commit is even made (see BestMethods BM-24)

**Notes:**  
History scrubbing was treated as hygiene, not the primary fix — rotation is what actually neutralizes a leaked credential, regardless of how "buried" it later becomes in git history.

---

### ISSUE-018 — Email History Gave No Visual Indication of Which Activity a Row Belonged To, and Its Found-Count Ignored Active Filters
**Severity:** Was 🟡 Medium  
**Status:** Fixed (2026-08-19) — confirmed via screenshot showing correct Activity badges and a filtered count ("18 of 122 email(s) shown")  
**Filed:** 2026-08-19

**Description:**  
User reported Email History rows (especially GroupMe messages) as unclear which team/sport they belonged to, even with the Activity filter pill set. Investigated directly rather than assumed: the `activityId` tagging and client-side filtering were both already correct (`renderHistory()` was genuinely filtering by `activeActivityFilter`) — the actual gap was that rows only ever displayed the sender `app` badge, never the resolved Activity name, and the "N email(s) found" status text was set once from the unfiltered total at load time and never updated when the Activity pill changed. A correctly-filtered list that never says so, and never shows *what* it's filtered to, reads as broken even when it isn't.

**Fix Applied:**  
- `renderHistory()` now shows a second badge resolving `e.activityId` to the Activity's name (or "(unassigned)").
- The found-count now reads `"{filtered} of {total} email(s) shown"` whenever the active filters narrow the list, instead of always showing the unfiltered total.

**See:** part of the same session's card-header work (DecisionLog PD-07) applying the same "show what this is actually about" principle.

---

### ISSUE-019 — GroupMe Email History Rows Always Labeled Literal "GroupMe" Instead of the Configured Group Label
**Severity:** Was 🟢 Low  
**Status:** Fixed (2026-08-19) — confirmed deployed  
**Filed:** 2026-08-19

**Description:**  
Email History showed every GroupMe-sourced row with the badge "GROUPME" regardless of which configured GroupMe group it came from — equivalent to showing "EMAIL" instead of a sender's configured label like "Coach Emmett." Root cause: `scanEmails()` uses each sender's configured `app` label via `detectApp()`, but `scanGroupMe()` hardcoded the literal string `'GroupMe'` instead of using that group's own configured `name` (Settings → GroupMe Groups → LABEL). Separately, the Settings "N sources" count on each Activity card only counted `senders.length`, silently omitting `groupmeGroups.length`.

**Fix Applied:**  
- GAS: `scanGroupMe()` writes `group.name || 'GroupMe'` instead of the literal string.
- Dashboard: Email History row coloring looks up a matched GroupMe group's own configured color (falling back to the old teal only for legacy rows still carrying the literal label); Settings source count sums `senders.length + groupmeGroups.length`.

---

### ISSUE-023 — 7-Day Schedule and Action Items Sections Loaded Expanded by Default
**Severity:** Was 🟢 Low (UX)  
**Status:** Fixed (2026-08-26)  
**Filed:** 2026-08-26

**Description:**  
The 7-Day Schedule and Action Items sections on Summary loaded expanded by default, inconsistent with the new collapsed-by-default behavior introduced for briefing cards (PD-11) in the same session.

**Fix Applied:**  
Default state changed to closed on page load, matching PD-11's card pattern.

---

### ISSUE-024 — Action Items Section Retained a Highlighted/Active Background After Being Collapsed
**Severity:** Was 🟢 Low (UX)  
**Status:** Fixed (2026-08-26)  
**Filed:** 2026-08-26

**Description:**  
After collapsing, the Action Items section kept a highlighted/active background. Root cause: a leftover active/focus style wasn't being cleared on collapse.

**Fix Applied:**  
Fixed in the same pass as ISSUE-023.

---

### ISSUE-025 — Calendars Tab Loaded 10 Events Successfully but Displayed None
**Severity:** Was 🟠 High  
**Status:** Fixed (2026-08-26)  
**Filed:** 2026-08-26

**Description:**  
After the AD-29 migration to `action=calendarEvents`, the Calendars tab reported a successful fetch (10 events loaded) but rendered nothing.

**Root cause (confirmed, not guessed):** Google Sheets auto-converts cell values that look like dates/times (e.g. "2026-08-26", "9:00 AM") into real `Date` objects on write, so the new endpoint's `getValues()` returned JS `Date` objects instead of the plain strings that were originally written. `JSON.stringify` serialized these as full ISO timestamps (e.g. "2026-08-26T00:00:00.000Z"), which never string-matched the frontend's `isoDate(day)` comparison — so the fetch succeeded but every event silently failed the per-day filter with no visible error.

**Fix Applied:**  
The endpoint now detects `Date`-typed cells and reformats them to the exact expected string shape (`yyyy-MM-dd` / `h:mm a`) before returning.

**See:** BestMethods (new entry this session) for the general lesson this exposed about Sheets' auto-coercion behavior.

---

### ISSUE-026 — Calendars Week-Range Label Wrapped Awkwardly on Mobile
**Severity:** Was 🟢 Low  
**Status:** Fixed (2026-08-26)  
**Filed:** 2026-08-26

**Description:**  
`.cal-month-label` had a fixed width that didn't fit "August 23-29, 2026" at small font sizes.

**Fix Applied:**  
Responsive font-size/letter-spacing and flexible sizing.

---

### ISSUE-027 — "Today" Row Misaligned in the Calendars Week View
**Severity:** Was 🟢 Low  
**Status:** Fixed (2026-08-26)  
**Filed:** 2026-08-26

**Description:**  
The Today badge's `margin-left:auto` combined with its first position in flex order was dragging the entire row (badge + day name + date) right instead of just the badge.

**Fix Applied:**  
Gave the badge `order:1` so it moves to the end of the row independently, without touching the margin.

---

### ISSUE-028 — Calendars Day-Name Text Rendered Smaller Than the Date Text Next to It
**Severity:** Was 🟢 Low  
**Status:** Fixed (2026-08-26)  
**Filed:** 2026-08-26

**Description:**  
`.week-day-name` ("Wed"/"Thu") was `.65rem` vs. `.week-day-date`'s `1rem`, creating a mismatched size pairing.

**Fix Applied:**  
Matched the sizes.

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

- [ ] **Deploy only from `tjunker9@gmail.com`** — deploying from any other Google account silently breaks Gmail access (see ISSUE-016) with no error until Refresh Briefing is clicked
- [ ] Load Calendars → all teams appear with correct event counts
- [ ] Events show at correct local time (test with a known event)
- [ ] Refresh Briefing completes within 90 seconds
- [ ] Summary contains cards from multiple senders (not just one app)
- [ ] Child filter pill actually changes what's shown on Calendars, Email History, *and* Summary
- [ ] Add / edit / delete a Child in Settings and confirm it sticks after a page reload
- [ ] Email History shows emails from the expected date range, correctly attributed to the right child
- [ ] Settings tab: test connection shows ✓ Connected
- [ ] "Load Settings from Cloud" round-trips correctly in a fresh/incognito browser
- [ ] All .ics URLs still valid (LeagueApps, TeamSnap, Playmetrics tokens don't expire silently) — note: this now only affects the Calendars tab's own week view, not the briefing (see AD-23)
- [ ] Google Calendar scan: GAS Executions log shows "Google Calendar: matched X event(s)..." with X > 0 after a Refresh Briefing (see ISSUE-020, and SessionStarter.md's testing checklist for the full verification steps)
- [ ] A To-Do added on one device appears on another after a page reload, confirming AD-24's cross-device sync

---

## Session Notes

> Add notes when issues are opened, updated, or closed.

- **Session 1 (Mar 2026):** Tracker initialized. ISSUE-003 already fixed. ISSUE-001 is the highest priority open item.
- **Session — 2026-08-17:** ISSUE-002 fixed (GAS-side ICS fetch, see AD-10). Two new critical/high bugs found, root-caused, fixed, and verified with automated tests: ISSUE-008 (child add/delete silently broken by a DOM-shadowing collision) and ISSUE-009 (email history never re-tagged after Settings reorg). Added ISSUE-010 to track the new AI-dependent child-tagging feature as a monitoring item, not a guaranteed-correct one. ISSUE-001 remains the top open item and now has an added dimension (its hardcoded app list is stale against the Activity/sender model).
- **Session — 2026-08-17 (later session):** Major filter/navigation UX rework (Category→Child→Activity cascade, Settings accordion, global scan window) shipped no new bugs on its own. Separately found and fixed ISSUE-011 (stale briefing content) and, as a direct side effect of that fix, ISSUE-012/ISSUE-013 (7-Day Schedule day-grouping and the calendar dateISO bug it exposed) — both root-caused by tracing the actual data flow, not guessed. Added the persistent To-Do feature, then caught and fixed ISSUE-014 (add-to-do not rendering) in the very next follow-up. Fixed ISSUE-015 (tab-nav phantom scrollbar), a real CSS spec quirk. Root-caused and resolved ISSUE-016 ("Gmail operation not allowed") to a wrong-account deployment via elimination (manual editor run reproduced it with no re-auth prompt; fresh consent didn't fix it) rather than guessed. ISSUE-001, ISSUE-004, ISSUE-005, ISSUE-006, ISSUE-007 remain open/deferred, untouched — ISSUE-006 in particular was explicitly confirmed NOT incidentally fixed by the Email History rewrite.
- **Session — 2026-08-18:** Found and fully closed ISSUE-017 (leaked Anthropic API key) end-to-end: rotate → migrate to Script Properties (Decision Log AD-21) → git history scrub → prevention recommendation logged. GroupMe backend integration applied to `SportsEmailScanner.gs`. PWA manifest/service worker/dashboard UI changes drafted, currently uncommitted. ISSUE-001, ISSUE-004, ISSUE-005, ISSUE-006, ISSUE-007 remain open/deferred, untouched.
- **Session — 2026-08-19:** Closed ISSUE-018 (Email History Activity display + stale found-count) and ISSUE-019 (GroupMe label/color/source-count), both confirmed. Added ISSUE-020 (Monitoring) for the new Google Calendar matching heuristic introduced by DecisionLog AD-23, which fully replaced the `.ics` pipeline feeding the briefing this session. ISSUE-001, ISSUE-004, ISSUE-005, ISSUE-006, ISSUE-007 remain open/deferred, untouched.
- **Session — 2026-08-21:** Ran the full Google Calendar scan testing
  checklist from the prior session to completion — confirmed working, one
  real issue found and root-caused (ISSUE-020 addendum). Found and closed
  ISSUE-022 (stale failing trigger, deleted directly, no code change).
  Opened ISSUE-021 (relative-date resolution bug), fix prompted, not yet
  deployed. ISSUE-001, ISSUE-004, ISSUE-005, ISSUE-006 remain open/
  deferred, untouched.
- **Session — 2026-08-26:** Full UI/PWA polish pass on Summary and Calendars. Fixed ISSUE-023/024 (7-Day Schedule and Action Items not matching the new collapsed-by-default card behavior, and a stuck highlight style). Fixed ISSUE-025 (Calendars tab loading events but displaying none — Sheets auto-coercing date/time strings to `Date` objects) and three small Calendars visual bugs (ISSUE-026/027/028). Logged ISSUE-029 (Activity color picker, deferred to the Settings redesign) and ISSUE-030 (dead `.ics` backend code, safe to remove later). ISSUE-001, ISSUE-004, ISSUE-005, ISSUE-006, ISSUE-021 remain open/deferred, untouched.
