# HQ Dashboard — Project Roadmap

**Last Updated:** 2026-08-27  
**Project:** Family sports + school activity intelligence dashboard  
**Current Phase:** Phase 3 complete (pivoted); Phase 4 mostly delivered. Phase 5's Google Calendar sync item is done and now fully verified end-to-end (AD-23, confirmed 2026-08-21), and the Calendars tab now reuses that same scanned data too (AD-29, 2026-08-26). Summary's mobile+desktop responsive/typography overhaul is done (PD-09 through PD-12, DD-04, 2026-08-26). A Sports/School accent color system now spans Summary/Calendars/Email History (PD-13), with Email History (PD-14) and To-Do (PD-15) both fully redesigned and deployed, and the dead Manual Updates tab removed entirely (PD-16, closes ISSUE-001). Daily Briefing refresh reliability fixed on mobile (AD-30, closes ISSUE-004) and a two-part sender-attribution bug fixed (AD-31, closes ISSUE-035) — both 2026-08-27. A Settings redesign (family-member grouping, editable/deletable sources, Activity color picker (ISSUE-029), general layout modernization) is Tom's stated next focus — note the underlying `calMatchName`/archiving/family-name fields it partly depends on (AD-25/26/27) and the daily-trigger parity fix (AD-28) were confirmed already deployed during 2026-08-27's doc audit, despite being previously logged "not yet deployed"; only the UI/layout pass itself remains undone. GroupMe integration and Phase 6 (PWA installability) remain in progress.

---

## Vision

A mobile-first progressive web app (PWA) that gives parents a single daily briefing covering all their kids' sports and school activities. AI-powered summarization (via Claude) turns a flood of app emails and calendar noise into clear priorities, timelines, and action items.

---

## Phase 1 — Sports HQ MVP ✅ COMPLETE

The baseline system is live and working.

- Gmail scanner via Google Apps Script (GAS)
- Claude AI summarization (priority card, cards, timeline, actions)
- .ics calendar parsing with team filters and weekly view
- Email history browsing with per-app filtering
- ~~Manual update input as fallback (still not wired to the prompt)~~ — removed entirely 2026-08-27 rather than wired up; auto-scan (Gmail + Google Calendar) already covers what this was meant to provide. Closed IssuesTracker ISSUE-001 by removal, see DecisionLog PD-16.
- Settings: Web App URL + team calendar subscriptions (localStorage)
- Daily 6 AM auto-briefing trigger

---

## Phase 2 — Polish & Reliability 🔧 IN PROGRESS

Incremental improvements to the existing Sports HQ system.

| Item | Priority | Status |
|------|----------|--------|
| PWA manifest + service worker (installable) | High | Pending |
| Loading skeletons / better empty states | Medium | Pending |
| Error handling improvements (clearer messages) | Medium | Pending |
| "Last refreshed" staleness indicator | Low | Pending |
| Manual update content included in Claude prompt | Medium | ✅ Resolved (2026-08-27), by removal not by building it — Manual Updates tab confirmed dead UI (its textareas were never read anywhere) and removed entirely rather than wired into the prompt. See IssuesTracker ISSUE-001, DecisionLog PD-16 |
| Sender whitelist UI in Settings (no code edit needed) | Medium | ✅ Done — delivered as part of the Child→Activity Settings rebuild (Decision Log AD-12) |
| Filter navigation cleanup (single Category→Child→Activity cascade) | Medium | ✅ Done — replaced separate per-sender (Email History) and per-calendar-name (Calendars) filter rows with one shared cascade (PD-05) |
| Settings Activities: collapse to reduce page length | Medium | ✅ Done — accordion cards, collapsed by default, new activities auto-expand |
| Days Back / Days Forward: global instead of per-Activity | Low | ✅ Done — one "Scan & Briefing Window" setting, with migration from old per-activity values (AD-15) |
| Briefing excludes already-passed content | High | ✅ Done — see IssuesTracker ISSUE-011, DecisionLog AD-16 |
| 7-Day Schedule shows all 7 days, grouped clearly | Medium | ✅ Done — see IssuesTracker ISSUE-012/013 |
| Interactive To-Do list | Medium | ✅ Done — persistent tab, synced across browsers via GAS (AD-24, closes out AD-18). Extended 2026-08-27 with optional due dates, Overdue/Upcoming/No due date/Done grouping, and click-to-edit in place (PD-15) |
| Refresh Briefing wait time / timeout handling | Medium | ✅ Resolved (2026-08-27) — see IssuesTracker ISSUE-004, DecisionLog AD-30. Not the originally-planned polling pattern; superseded by splitting the refresh into 4 independently-retryable steps with a real progress bar |
| Settings redesign — family members grouped by Sport/School, existing-vs-new activity flow, editable/deletable sources, archive instead of delete, Activity color picker (ISSUE-029), general layout modernization to match the new typography/design system | High | Not started. Note: DecisionLog AD-25 (`calMatchName`)/AD-26 (Activity archiving) — the backend/data-model fields this row's "archive instead of delete" and calendar-matching scope depend on — were confirmed already deployed during a 2026-08-27 doc audit (previously mis-tracked as "not yet deployed"); the UI/layout modernization itself, including the color-picker question (ISSUE-029) and general visual consistency, is still fully open and is Tom's stated next focus |
| Summary page mobile+desktop responsive/typography overhaul — filter/nav layout (PD-09/PD-10), typography system (DD-04), collapsible briefing cards (PD-11), Calendars migrated to the Google Calendar-scanned data source (AD-29) | High | ✅ Done (2026-08-26) — committed, deployed, confirmed working by Tom via live testing |
| Verify Email History, To-Do, and Manual Updates tabs against the new typography/responsive design system | Medium | ✅ Done (2026-08-27) for Email History and To-Do — both fully redesigned (PD-14/PD-15), not just verified. Manual Updates is moot — removed entirely rather than verified (see above) |
| Sports/School accent color system — blue/amber driven by the category switch, applied across Summary/Calendars/Email History chrome; To-Do deliberately excluded in favor of a fixed green | High | ✅ Done (2026-08-27) — PD-13 (system + Summary/Calendars/Email History), PD-15 (To-Do's fixed-green exclusion) |
| Email History redesign — family-member color instead of source-app/activity badges | Medium | ✅ Done (2026-08-27) — PD-14 |
| Sender-attribution accuracy — shared-platform-domain senders (e.g. multiple SportsEngine orgs) mis-attributed to the wrong Activity | High | ✅ Done (2026-08-27) — DecisionLog AD-31, two-part fix (match specificity, then From+Subject search). Tom still needs to add a `top center lacrosse` sender entry in Settings for his specific TC Lax case — see AD-31 |

---

## Phase 3 — School HQ Dashboard ✅ COMPLETE (pivoted to Unified architecture)

**This phase shipped, but not the way it was originally scoped below.** The original plan called for a fully separate `school-dashboard.html` + `SchoolEmailScanner.gs` + `School HQ` Sheet. That was reversed mid-project — see Decision Log **PD-03**. What actually got built:

- School is a second `category` (`sports` / `school`) inside the *same* `sports-dashboard.html` and the *same* `SportsEmailScanner.gs`, writing to the *same* spreadsheet with separate per-category sheet tabs (`SchoolEmails`/`SchoolEmailHistory`/`SchoolSummary`/`SchoolCalendarEvents` alongside the original Sports tabs)
- Same design system, same Sports/School switcher pattern originally envisioned for 3c's "unified shell" — since there was only ever one shell, that sub-phase is effectively moot
- Claude prompt has a school-specific variant (`buildSchoolPrompt`) tuned for assignments/tests/deadlines rather than games

Original sub-phase notes, kept for reference:

<details>
<summary>Original Phase 3 plan (superseded)</summary>

### 3a — Backend (SchoolEmailScanner.gs)
- Gmail scanner for school-related senders (teachers, school domain, PTA, etc.)
- Configurable sender list (school email domains + specific addresses)
- Assignment/test/event extraction via Claude
- Deadline-aware prioritization (due today, due this week, upcoming)
- Google Sheet tabs: SchoolEmails, SchoolHistory, SchoolSummary, SchoolCalendar

### 3b — Frontend (school-dashboard.html or merged tab)
- Same design system as sports dashboard
- Summary card layout adapted for academics (subject-coded cards)
- "Due This Week" timeline view (vs. sports' game schedule view)
- Subject filter buttons (Math, Science, English, etc.)
- Assignment detail panel

### 3c — Shared Navigation
- Unified shell with Sports HQ / School HQ top-level tabs or separate installable apps
- Shared Settings (single Web App URL if merged GAS, or separate URLs)

</details>

---

## Phase 4 — Kid Profiles 🔧 MOSTLY DELIVERED

This was originally scoped as a future phase, but landed early and mostly complete as a side effect of rebuilding Settings around Decision Log **AD-12** (Child → Activity → Sender/Calendar linking model).

| Original goal | Status |
|------|--------|
| Profile switcher (e.g., "Jack" / "Emma" / "All Kids") | ✅ Done — global Child filter pill row (All / per-child) above the Sports/School switch |
| Per-kid color coding throughout | 🟡 Partial — children and activities both carry a color, used in the child pills, chip pickers, and calendar dots; not yet audited for full consistency everywhere a kid could visually be represented |
| Per-kid calendar subscriptions | ✅ Done — calendars link to an Activity, which links to one or more children |
| Merged family timeline that shows all kids' events together | ✅ Done — the "All" child filter already shows everyone's events together in one weekly view; a dedicated cross-kid timeline widget was never built separately, but the need is met |
| Claude prompt adapted to mention child names in briefings | ✅ Done, and taken further — cards/timeline/actions now carry a structured `childIds` field (not just prose mentions), so the child filter also works on the Summary tab. See ISSUE-010 for the caveat that this is model output, not guaranteed-deterministic; the same caveat now also applies to `dateISO` (see ISSUE-011/013) and, as of 2026-08-19, `activityId` (PD-06, ISSUE-020) — the Activity filter now works on Summary too, not just Calendars/Email History. |
| Third filter tier: which specific team/school | ✅ Done — Activity added as the third cascade level (Category → Child → Activity), replacing the old separate per-sender/per-calendar filter rows entirely (PD-05) |

**Remaining for a clean close-out:** a short polish pass confirming color consistency across every kid-related UI element, and deciding whether a dedicated "family timeline" widget is still wanted now that "All" filter effectively covers it.

---

## Phase 5 — Enhanced Data Sources 📡

Richer inputs beyond Gmail.

| Feature | Notes | Status |
|---------|-------|--------|
| Google Calendar sync (direct API) | Replaces `.ics` as the briefing's calendar data source entirely, using GAS's native `CalendarApp` — a further step past AD-10 (which moved `.ics` fetching server-side but kept `.ics` itself as the source) | ✅ Done — see DecisionLog AD-23. Scope note: this only replaced the source feeding the *briefing*; the Calendars tab's own week-view UI is still `.ics`-based and untouched — see the new backlog item below for that follow-up |
| GroupMe API integration | Pull messages without email delay | 🔧 In progress — messages normalized into the existing history sheet, zero prompt changes needed (AD-20); GAS backend (`scanGroupMe()`) and Settings UI drafted, currently uncommitted. Blocked on completing GroupMe's OAuth app-registration flow to obtain a real token + group ID for end-to-end testing |
| Push notifications / SMS | Same-day urgent alerts | Not started |
| Weather injection on game days | AccuWeather or NWS free API | Not started |
| Carpool coordination view | Built from existing Carpool Kids emails | Not started — flagged as the one Phase 5 item likely to need its own UI design pass rather than slotting into the existing card/Activity model |
| Score/standings auto-fetch | GameChanger or SportsEngine webhook | Not started |

---

## Phase 6 — PWA + Distribution 📱

Make it properly installable and shareable from the same single URL on both desktop and mobile.

| Item | Status |
|------|--------|
| PWA manifest (`manifest.json`) | 🔧 Drafted, uncommitted |
| Service worker (`sw.js`) — installability + safe offline fallback, deliberately network-first for the dashboard itself, not full offline caching (AD-22) | 🔧 Drafted, uncommitted |
| Dashboard `<head>` wiring (manifest link, theme-color, Apple touch icon, SW registration) | Pending |
| Real branded icon assets (180px Apple touch, 192px, 512px, plus maskable variants) | Pending — no icon art exists yet |
| Capacitor wrapper for iOS/Android app store submission (PWABuilder path) | Not started |
| Multi-family distribution (Model A — self-hosted per family) | Decided as the starting approach; setup guide not yet written |

---

## Backlog / Ideas (Unscheduled)

- Dark mode toggle
- Exportable weekly family schedule PDF
- Voice briefing (text-to-speech of daily summary)
- Household task integration (chores tied to sports schedule)
- Nutrition/hydration reminders on game days
- Photo log for game memories
- ~~Rework the Manual Updates tab to be Activity-driven instead of a hardcoded app list, and actually wire it into the prompt (folds ISSUE-001 and the stale-tab problem into one pass)~~ — moot as of 2026-08-27; the tab was removed entirely instead (PD-16), not reworked.
- Now that Calendars no longer uses each Activity's own `color` field for its event bar (removed 2026-08-27 — bars follow the Sports/School accent instead, see PD-13/PD-14 and IssuesTracker ISSUE-029), decide during the upcoming Settings redesign whether that color-picker field is still worth keeping in the Activity card UI at all, or should be dropped alongside the redesign.
- ~~Extend the Calendars tab to reuse Google-Calendar-scanned data instead of maintaining a separate `.ics` pipeline (user-requested, 2026-08-19)~~ — ✅ **Done 2026-08-26**, see DecisionLog AD-29. Note: this closed the *data-source* consolidation only; the day/kid/sport filtering itself already existed before this session and simply carried over onto the new data source unchanged. A further "show more than 7 days out" extension is not part of AD-29 and remains a distinct, still-unscoped idea if wanted later.
- Remove dead `.ics`-subscription backend code (`fetchIcsServerSide()`/`action=ics` handler, `cals` config field) once the new Calendars data path (AD-29) has proven stable for a while — see IssuesTracker ISSUE-030. Low priority, not urgent.
- Personalization and/or multi-activity support as a paid tier for
  other families running their own self-hosted deployment (user idea,
  2026-08-21) — no design or commitment yet; captured alongside the
  new editable Family Name field (AD-27), which is a plain,
  unrestricted building block for this either way.

---

## Session Notes

> Add notes here after each session about what shifted in the roadmap.

- **Session 1 (Mar 2026):** Established roadmap. Phase 1 confirmed complete.
- **Session — 2026-08-17:** Phase 3 marked complete, with a documented architectural pivot (unified dashboard instead of separate School system — Decision Log PD-03). Phase 4 (Kid Profiles) turned out to be mostly delivered as a side effect of the same work; marked accordingly with an honest partial on per-kid color-consistency polish. Phase 2's sender-whitelist item is done. Manual Updates (ISSUE-001) remains the most notable unfinished Phase 1/2 item, and now has an extra wrinkle (its hardcoded app list doesn't match the new Activity/sender model) worth fixing in the same pass.
- **Session — 2026-08-17 (later session):** Significant Phase 2 UX pass: unified filtering into one Category→Child→Activity cascade (also closes out the last piece of Phase 4's "which team/school" filtering), collapsed Settings Activities into an accordion, moved scan window to a global setting. Separately, fixed a real accuracy problem — the briefing was surfacing already-past content from old emails — plus the day-grouping fix that came out of that work and the calendar-date bug it exposed along the way. Added a persistent To-Do list as a new small feature (not originally scoped, but a direct, low-cost extension of the Action Items work). Deliberately deferred the Refresh Briefing timeout/speed question (ISSUE-004) to its own future session rather than rushing a fix. Also resolved a deployment/process issue (ISSUE-016, wrong Google account) that isn't roadmap-relevant on its own but is now documented as a standing operational rule in BestMethods (BM-19) and DecisionLog (AD-19).
- **Session — 2026-08-18:** Started Phase 5 (GroupMe integration — backend + Settings UI drafted, blocked on live OAuth credentials) and Phase 6 (PWA manifest + service worker drafted, dashboard wiring and icons still pending). Confirmed the multi-family distribution direction: Model A (self-hosted per family) over a centralized SaaS rebuild, given Google's OAuth verification requirements for the alternative. Separately resolved a critical security incident (leaked Anthropic API key, IssuesTracker ISSUE-017) unrelated to roadmap progress but worth noting here since it delayed feature work this session.
- **Session — 2026-08-19:** Closed out Phase 5's Google Calendar sync item (AD-23) — a user-requested architecture change (drop per-team `.ics` maintenance in favor of a direct Google Calendar scan), not originally scoped work. Phase 4's Activity-filter gap on Summary (parallel to the childIds gap already closed) fixed via PD-06. Phase 2's To-Do list item upgraded from localStorage-only to GAS-synced (AD-24, closes AD-18). Added a new backlog item for a longer-range, kid/sport/school-filterable Calendars tab, explicitly deferred rather than built this session. GroupMe (Phase 5) remains blocked on OAuth setup, untouched this session aside from an unconfirmed label/count fix — see IssuesTracker ISSUE-019.
- **Session — 2026-08-21:** Ran the Google Calendar scan testing
  checklist from the prior session to full completion — confirmed
  working, with one real calendar-matching bug found and root-caused
  along the way (see IssuesTracker ISSUE-020's addendum). Also found
  and closed a stale, silently-failing trigger (ISSUE-022) and opened
  a new relative-date-resolution bug (ISSUE-021). Designed a
  significant Settings redesign with Tom via interactive mockups
  (family-member terminology, Sport/School grouping, editable
  sources, activity archiving, a calendar/display name split, and a
  personalizable family name) and closed a gap where the 6 AM daily
  trigger skipped the calendar scan entirely (AD-28) — all written up
  as 8 CC prompts this session, **none deployed yet.**
- **Session — 2026-08-26:** Full UI/PWA polish pass, all deployed and confirmed working: Summary's filter/nav layout overhaul (PD-09/PD-10), collapsible briefing cards (PD-11), and a project-wide typography swap to Quicksand/Nunito Sans (DD-04, supersedes DD-01). Marked the "extend Calendars tab" backlog item done — Calendars now reads scanned Google Calendar data via a new endpoint (AD-29) instead of live `.ics` subscriptions, which were removed from Settings (PD-12). Six small bugs found and fixed along the way (ISSUE-023 through ISSUE-028); logged an Activity color-picker gap (ISSUE-029) into the upcoming Settings redesign's scope, and a dead-code cleanup item (ISSUE-030). Added new backlog items for verifying Email History/To-Do/Manual Updates against the new design system and for the eventual `.ics` code removal.
- **Session — 2026-08-27:** Closed out the "verify Email History/To-Do/
  Manual Updates" item from last session — Email History and To-Do were
  fully redesigned rather than just verified (PD-14, PD-15), and Manual
  Updates was removed entirely rather than verified, closing ISSUE-001
  in the process. Added the Sports/School accent color system (PD-13).
  Resolved the long-open "Refresh Briefing timeout" item (ISSUE-004,
  originally scoped for a polling pattern) via a different, better fix —
  splitting the refresh into 4 independently-retryable steps (AD-30).
  Fixed a two-part sender-attribution bug affecting shared-platform-domain
  senders (AD-31). Doc-audit finding, not new work: AD-25/AD-26 (and
  AD-27/AD-28, tracked in DecisionLog/TechnicalArchitecture rather than
  here) were mis-tracked as "not yet deployed" and are actually already
  live — corrected in this pass; the Settings redesign row above is
  updated to reflect that its remaining scope is the UI/layout pass, not
  the underlying fields. **Same-day follow-up:** reconciled this pass
  against a separately-drafted set of doc-update prompts that hadn't seen
  it yet — updated the `**Current Phase:**` header and struck through the
  Phase 1 Manual Updates bullet and the stale "rework Manual Updates"
  backlog item (all three were missed in the first pass), added a backlog
  item questioning the Activity `color` field's future, and filed the
  sender-attribution bug as IssuesTracker ISSUE-035 (the other prompt's
  proposed ISSUE-031/032 numbers collided with different bugs already
  filed there this session).
