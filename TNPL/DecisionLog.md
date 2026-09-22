# TNPL — Decision Log

Prefix: TP

| ID | Date | Decision | Rationale |
|---|---|---|---|
| TP-001 | 2026-09-22 | Stack: React + Vite + Firebase | Matches the UP Golf / Club Golf PWA pattern Tom already uses and knows |
| TP-002 | 2026-09-22 | GitHub repo `TNPL`, private, under `Whit19` | Follows existing DataForge repo-naming convention (e.g. AFAS, up-golf-pwa) |
| TP-003 | 2026-09-22 | Notion Client tag: new "Paddlers" option added | No existing Client option fit the TNPL player group; parallels the existing "Golfers" tag used for UP Golf |
| TP-004 | 2026-09-22 | v1 scope = full weekly loop (availability, pairing, scoring, Elo) | Tom chose to build the whole loop now rather than staging it incrementally |
| TP-005 | 2026-09-22 | Players can play up to 2 matches/week, as 2 independent match-groups (different partners/opponents possible) | Confirmed by Tom; drives `matchNumber` field and second pairing pass |
| TP-006 | 2026-09-22 | Change requests (swap-out/time-change) get an approve/deny flow in v1 | Tom wants this from day one, not deferred |
| TP-007 | 2026-09-22 | Time-slot structure: 3 fixed slots (6:00/7:15/8:30 PM), 2 groups (courts) per slot | Confirmed from live `Fall_Paddle_2026_Session_2_Copy.xlsx` workbook |
| TP-008 | 2026-09-22 | No locked-partner/couples pairing constraint | The `Couples` sheet in the sample workbook was a one-off event, not part of regular Thursday pairing |
