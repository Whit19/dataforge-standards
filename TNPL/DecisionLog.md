# TNPL — Decision Log

Prefix: TP

| ID | Date | Decision | Rationale |
|---|---|---|---|
| TP-001 | 2026-09-22 | Stack: React + Vite + Firebase | Matches the UP Golf / Club Golf PWA pattern Tom already uses and knows |
| TP-002 | 2026-09-22 | GitHub repo `TNPL`, private, under `Whit19` | Follows existing DataForge repo-naming convention |
| TP-003 | 2026-09-22 | Notion Client tag: new "Paddlers" option added | No existing Client option fit; parallels "Golfers" tag used for UP Golf |
| TP-004 | 2026-09-22 | v1 scope = full weekly loop (availability, pairing, scoring, Elo) | Tom chose to build the whole loop now rather than staging it |
| TP-005 | 2026-09-22 | Players can play up to 2 matches/week, as 2 independent match-groups | Confirmed by Tom; drives `matchNumber` field and second pairing pass |
| TP-006 | 2026-09-22 | Change requests get an approve/deny flow in v1 | Tom wants this from day one, not deferred |
| TP-007 | 2026-09-22 | Time-slot structure: 3 fixed slots, 2 groups (courts) per slot | Confirmed from live `Fall_Paddle_2026_Session_2_Copy.xlsx` workbook |
| TP-008 | 2026-09-22 | No locked-partner/couples pairing constraint | The `Couples` sheet was a one-off event, not regular Thursday pairing |
| TP-009 | 2026-09-22 | `seasonEnrollment` modeled separately from the persistent player roster | Not all players opt in each season; roster persists, participation doesn't |
| TP-010 | 2026-09-22 | Auth: Google + Email Link providers, both mapped to `players` by email | Google-only would exclude players without a Google account |
| TP-011 | 2026-09-22 | Scores editable (last write wins) until admin locks the week | Easy correction of mis-entered scores, not a hard first-submission lock |
| TP-012 | 2026-09-22 | Elo computed only at "Lock Week" (admin action), not as scores are entered | Keeps Elo stable until the admin confirms results are final |
| TP-013 | 2026-09-22 | Added `role: 'player' \| 'staff'` to `players` instead of a separate collection | Club workers need schedule visibility but aren't players; reuses auth-matching logic |
| TP-014 | 2026-09-22 | Added `playerLinks/{authUid} -> playerId` reverse-index collection | Firestore security rules can only `get()` by known path, not query; needed to resolve caller identity for `isAdmin()`/`isPlayerRole()` rule checks without custom claims (CC, implementing TP-010/013) |
| TP-015 | 2026-09-22 | **Open — not yet decided.** Mid-season `seasonEnrollment` opt-out: a player can flip `optedIn` to `false` at any time (rules allow self-write anytime); what should happen to weeks/matchGroups already built from their availability once they opt out mid-season is undecided | Tom flagged this as unresolved when confirming self-service opt-in write access; deferred until the opt-in UI / pairing engine phase, needs a decision before that UI ships |
