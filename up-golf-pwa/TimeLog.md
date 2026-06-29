# TimeLog.md — UP Golf PWA
**Session time log**

---

| Date | Duration | Summary |
|------|----------|---------|
| 2026-06-29 | ~4 hrs | Phase 8A: historical data import (1,521 docs across 4 history_ collections, 2011–2025), off-season 3-tab nav, HistoryScreen.jsx (Tournaments + Players tabs), Tabler Icons CDN, app renamed Tourney Golf, Firestore rules + composite indexes for history_ collections, AdminArchiveTournament.jsx, "Full Results" button in history detail. |
| 2026-06-04 | ~1.5 hrs | Added in-app PDF guides (App Setup + User Guide) to Info screen via Firebase Storage; resolved laptop dev environment setup; called Phase 7 complete. |
| 2026-06-13 | ~1 hr | Post-freeze exception (DEC-136): fixed unreachable Save button on "Add Dining Event" form on iOS PWA (ISS-136) by converting AnnouncementModal/DiningEventModal to inline forms after two reverted CSS-only attempts; modal CSS and useModalScrollLock fully reverted/removed. |
| 2026-06-16 | ~2.5 hrs | Added Player Profile card + Players Directory feature post-freeze; Storage rules updated; firstYear bulk-set on all 32 player docs. |
| 2026-06-21 | ~45 min (approx — confirm/adjust) | Post-freeze fix (DEC-139/ISS-140): diagnosed and fixed players unable to see live Skins standings on Money screen after R1 — root-caused to a firestore.rules payments read restriction silently blocking the broad query useSkins.js needs; opened payments read to all signed-in players, write access unchanged. Verified working by Tom with a player login. |
| 2026-06-24 | ~4 hrs | Phase 8 kickoff: recovered corrupt historical Excel, built UP_Golf_Historical_Import_v3.xlsx (6 sheets, 2011–2025, audited final), built TournamentResultsPDF component (admin export + player history view), added History card to Info screen, migrated session docs to public GitHub URL fetch workflow, updated MASTER_CLAUDE_PROTOCOL.md with CC prompt delivery + GitHub fetch standards. |
