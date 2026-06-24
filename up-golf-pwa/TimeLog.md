# TimeLog.md — UP Golf PWA
**Session time log**

---

| Date | Duration | Summary |
|------|----------|---------|
| 2026-06-04 | ~1.5 hrs | Added in-app PDF guides (App Setup + User Guide) to Info screen via Firebase Storage; resolved laptop dev environment setup; called Phase 7 complete. |
| 2026-06-13 | ~1 hr | Post-freeze exception (DEC-136): fixed unreachable Save button on "Add Dining Event" form on iOS PWA (ISS-136) by converting AnnouncementModal/DiningEventModal to inline forms after two reverted CSS-only attempts; modal CSS and useModalScrollLock fully reverted/removed. |
| 2026-06-16 | ~2.5 hrs | Added Player Profile card + Players Directory feature post-freeze; Storage rules updated; firstYear bulk-set on all 32 player docs. |
| 2026-06-21 | ~45 min (approx — confirm/adjust) | Post-freeze fix (DEC-139/ISS-140): diagnosed and fixed players unable to see live Skins standings on Money screen after R1 — root-caused to a firestore.rules payments read restriction silently blocking the broad query useSkins.js needs; opened payments read to all signed-in players, write access unchanged. Verified working by Tom with a player login. |
