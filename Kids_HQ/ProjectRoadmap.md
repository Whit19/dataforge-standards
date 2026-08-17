# HQ Dashboard — Project Roadmap

**Last Updated:** August 2026  
**Project:** Family sports + school activity intelligence dashboard  
**Current Phase:** Phase 3 complete (pivoted); Phase 4 mostly delivered as part of the same work — see notes below.

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
- Manual update input as fallback (still not wired to the prompt — see IssuesTracker ISSUE-001)
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
| Manual update content included in Claude prompt | Medium | Pending — ISSUE-001 |
| Sender whitelist UI in Settings (no code edit needed) | Medium | ✅ Done — delivered as part of the Child→Activity Settings rebuild (Decision Log AD-12) |

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
| Claude prompt adapted to mention child names in briefings | ✅ Done, and taken further — cards/timeline/actions now carry a structured `childIds` field (not just prose mentions), so the child filter also works on the Summary tab. See ISSUE-010 for the caveat that this is model output, not guaranteed-deterministic. |

**Remaining for a clean close-out:** a short polish pass confirming color consistency across every kid-related UI element, and deciding whether a dedicated "family timeline" widget is still wanted now that "All" filter effectively covers it.

---

## Phase 5 — Enhanced Data Sources 📡

Richer inputs beyond Gmail. Unchanged from original plan — none of this has started.

| Feature | Notes |
|---------|-------|
| Google Calendar sync (direct API) | Replace .ics proxy workaround — note the proxy workaround itself was already replaced by a GAS-side fetch (AD-10); this would be a further step (direct API vs. ICS feed at all) |
| GroupMe API integration | Pull messages without email delay |
| Push notifications / SMS | Same-day urgent alerts |
| Weather injection on game days | AccuWeather or NWS free API |
| Carpool coordination view | Built from existing Carpool Kids emails |
| Score/standings auto-fetch | GameChanger or SportsEngine webhook |

---

## Phase 6 — PWA + Distribution 📱

Make it properly installable and shareable. Unchanged from original plan — not started. Currently hosted as a static file (GitHub Pages), which covers "shareable" but not "installable."

- PWA manifest, icons, splash screens
- Service worker for offline summary caching
- Capacitor wrapper for iOS/Android app store submission (PWABuilder path)
- Optional: share with other parents (multi-user GAS deployment)

---

## Backlog / Ideas (Unscheduled)

- Dark mode toggle
- Exportable weekly family schedule PDF
- Voice briefing (text-to-speech of daily summary)
- Household task integration (chores tied to sports schedule)
- Nutrition/hydration reminders on game days
- Photo log for game memories
- Rework the Manual Updates tab to be Activity-driven instead of a hardcoded app list, and actually wire it into the prompt (folds ISSUE-001 and the stale-tab problem into one pass)

---

## Session Notes

> Add notes here after each session about what shifted in the roadmap.

- **Session 1 (Mar 2026):** Established roadmap. Phase 1 confirmed complete.
- **Session — 2026-08-17:** Phase 3 marked complete, with a documented architectural pivot (unified dashboard instead of separate School system — Decision Log PD-03). Phase 4 (Kid Profiles) turned out to be mostly delivered as a side effect of the same work; marked accordingly with an honest partial on per-kid color-consistency polish. Phase 2's sender-whitelist item is done. Manual Updates (ISSUE-001) remains the most notable unfinished Phase 1/2 item, and now has an extra wrinkle (its hardcoded app list doesn't match the new Activity/sender model) worth fixing in the same pass.
