# HQ Dashboard — Project Roadmap

**Last Updated:** March 2026  
**Project:** Family sports + school activity intelligence dashboard  
**Current Phase:** Phase 1 — Sports HQ (functional MVP)

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
- Manual update input as fallback
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
| Manual update content included in Claude prompt | Medium | Pending |
| Sender whitelist UI in Settings (no code edit needed) | Medium | Pending |

---

## Phase 3 — School HQ Dashboard 📚 NEXT

Build a parallel system for school activities following the same architecture.

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

---

## Phase 4 — Kid Profiles 👧👦

Multiple children tracked separately with unified family view.

- Profile switcher (e.g., "Jack" / "Emma" / "All Kids")
- Per-kid color coding throughout
- Per-kid calendar subscriptions
- Merged family timeline that shows all kids' events together
- Claude prompt adapted to mention child names in briefings

---

## Phase 5 — Enhanced Data Sources 📡

Richer inputs beyond Gmail.

| Feature | Notes |
|---------|-------|
| Google Calendar sync (direct API) | Replace .ics proxy workaround |
| GroupMe API integration | Pull messages without email delay |
| Push notifications / SMS | Same-day urgent alerts |
| Weather injection on game days | AccuWeather or NWS free API |
| Carpool coordination view | Built from existing Carpool Kids emails |
| Score/standings auto-fetch | GameChanger or SportsEngine webhook |

---

## Phase 6 — PWA + Distribution 📱

Make it properly installable and shareable.

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

---

## Session Notes

> Add notes here after each session about what shifted in the roadmap.

- **Session 1 (Mar 2026):** Established roadmap. Phase 1 confirmed complete.

