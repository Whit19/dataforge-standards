# IssuesTracker.md — Club Golf App
**Tracks open issues, bugs, deferred items, and resolved items.**

---

## Issue Format
Each entry includes: ID, title, type, priority, status, description, and resolution (if resolved).

**Types:** Bug | Decision Needed | Feature Gap | Deferred | Architecture
**Priority:** P1 (blocking) | P2 (important, non-blocking) | P3 (nice to have)
**Status:** Open | In Progress | Resolved | Deferred

---

## Open Issues

| ID | Title | Type | Priority | Notes |
|----|-------|------|----------|-------|
| CGI-042 | Invite URL fragility in non-Safari email clients | Bug | P2 | Gmail/Outlook in-app browsers may mangle query params or block Firebase callables. Safari tip + plain-text URL added to email template as mitigation. Monitor after first wave of invites. |

---

## Deferred Items (known v2 scope)

| ID | Item | Notes |
|----|------|-------|
| CGI-D001 | GHIN handicap sync | Requires GHIN API credentials. Manual entry used in v1. |
| CGI-D002 | Full member roster auth | All members have accounts in v2. v1 uses invite flow. |
| CGI-D003 | Multi-course support | ✅ Resolved in Phase 2A (CGAD-036) |
| CGI-D004 | Betting / side games on 6-Point Game | Real-money or points-based betting layer. |
| CGI-D005 | Thursday Night Men's League | Requires league structure: seasons, weeks, flights, standings. |
| CGI-D006 | Additional game formats | Stableford, Nassau, Skins, Wolf, etc. |
| CGI-D007 | Multi-club admin dashboard | Central dashboard for managing multiple clubs (Phase 5). |
| CGI-D008 | Stripe / subscription billing | Payment infrastructure for per-club subscriptions (Phase 5). |
| CGI-D009 | Golf Genius / CSV roster import | ✅ Resolved — importMembers.cjs built and run 5/15/26. |
| CGI-D010 | Separate club admin vs. game admin roles | Single admin role in v1. |
| CGI-D011 | Guest player auto-save to roster | Currently per-game only. |
| CGI-D012 | Stats leaderboard page | ✅ Resolved — LeaderboardPage built 5/27/26 (Session 23). |
| CGI-D013 | Firestore role-based security rules (full) | Roles-based via clubs/{clubId}/roles/{uid} in v1. Full per-doc ownership rules deferred. |
| CGI-D014 | Contact picker for roster add | Browser Contact Picker API not supported on iOS Safari. Manual entry in v1. |
| CGI-D015 | Profile page — game history list | "My games" button navigates to filtered HistoryPage. Inline game list on profile deferred. |
| CGI-D016 | Settings page | Currently a "Coming soon" stub at /settings. Tab removed from bottom bar. |
| CGI-D017 | Season filtering for leaderboard | ✅ Resolved — per-year stats buckets + year picker built 5/27/26 (Session 23). |
| CGI-D018 | Course API integration | Allow admin to search/import course data instead of manual entry. Deferred to Phase 4H. |
| CGI-D019 | tnmlTeam migration to league roster | `tnmlTeam` field on member doc is temporary. Migrates to `leagues/{leagueId}/roster` in Phase 3. |
| CGI-D020 | Scheduled game — past-date auto-cleanup | Scheduled games with past dates remain on Scheduled tab indefinitely. Auto-move or admin cleanup deferred. |
| CGI-D021 | Transactional email provider upgrade | Switch from Gmail/Nodemailer to Resend or SendGrid for better deliverability. Tracked as CGI-041 until prioritized. |
| CGI-D022 | Completed games section on GamesPage | Last Game card shows most recent only. Full completed list is in History tab. Full completed section on GamesPage deferred. |
| CGI-D023 | Per-format stats history | LeaderboardPage shows overall stats only. Per-format breakdown (e.g. "my 6-Point record") deferred to Phase 4G. |
| CGI-D024 | Bulk HI update from Golf Genius export | No automated HI refresh flow. Currently manual or via one-off scripts. HI Age indicator surfaces staleness to organizer at game creation. |

---

## Resolved Issues

| ID | Title | Resolution |
|----|-------|------------|
| CGI-001 | Firestore composite index for activeGames query | Index created via Firebase console. Resolved 4/29/26. |
| CGI-002 | Settings review panel during game | ⚙ gear button + modal built. Resolved 5/5/26. |
| CGI-003 | Hole circle visual polish | Reviewed on phone — looks fine. Closed 5/6/26. |
| CGI-004 | HoleResultCard casket/bonus display | Updated with bonus rows, ★ Basket, ☠ Casket. Resolved 5/5/26. |
| CGI-005 | Bottom tab bar | BottomTabBar built. Resolved 5/12/26. |
| CGI-006 | Share live link sending localhost URL | Hardcoded production URL in ShareButton. Resolved 5/6/26. |
| CGI-007 | ViewerPage blank black page | games collection rule changed to allow read: if true. Resolved 5/6/26. |
| CGI-008 | Date showing one day ahead | Fixed to use local date not UTC. Resolved 5/6/26. |
| CGI-009 | Member invite + signup flow | Full flow built. Resolved 5/7/26. |
| CGI-010 | PWA icon not showing on home screen | Filename mismatch fixed. Resolved 5/7/26. |
| CGI-011 | Game creation restricted to admin only | Removed admin guards. Resolved 5/7/26. |
| CGI-012 | Deleting member left orphaned Firebase Auth account | deleteMember Cloud Function removes Auth account. Resolved 5/7/26. |
| CGI-013 | Landscape scorecard view | ScorecardView.jsx built and wired into ScoreEntryPage, ResultsPage, ViewerPage. Resolved 5/8/26. |
| CGI-014 | ResultsPage requires scrolling to reach share/back buttons | Back arrow + share icon moved to header. Resolved 5/8/26. |
| CGI-015 | ResultsPage back button always goes to /games | Context-aware navigation: returns to /history if navigated from HistoryPage. Resolved 5/8/26. |
| CGI-016 | GameCard share fires on tap anywhere in card bottom | gc-bottom stopPropagation added. Resolved 5/8/26. |
| CGI-017 | Player picker unusable as roster grows | Replaced select dropdown with modal picker sheet + real-time search filter. Resolved 5/11/26. |
| CGI-018 | Default settings not matching club preference | AUTO_RECRACK and casketEnabled:true set as defaults. Resolved 5/11/26. |
| CGI-019 | PH missing from Review step in game creation | PH computed inline and added to player detail row. Resolved 5/11/26. |
| CGI-020 | Step tabs not back-navigable in score entry | Crack/Prox/Scores/Result tabs now tappable to navigate back. Resolved 5/11/26. |
| CGI-021 | No stroke indicator during score entry steps | Persistent stroke chip row added below hole header on all 4 steps. Resolved 5/11/26. |
| CGI-022 | Scorecard text too small on iPhone | All font sizes increased; layout refactored to fill full viewport. Resolved 5/11/26. |
| CGI-023 | Firestore security rules too open | Roles-based rules implemented via clubs/{clubId}/roles/{uid}. Resolved 5/12/26. |
| CGI-024 | ProfilePage was Coming Soon stub | Full ProfilePage built: stats, favorites, preferences, logout confirmation. Resolved 5/12/26. |
| CGI-025 | No club leaderboard | Collapsible Club leaders card added to HistoryPage (later moved to dedicated LeaderboardPage in Session 23). Resolved 5/12/26. |
| CGI-026 | No way to filter history to own games | All/Mine toggle added to HistoryPage; My games button on ProfilePage. Resolved 5/12/26. |
| CGI-027 | No way to manage favorites | FavoritePicker modal on ProfilePage; star toggle on RosterPage. Resolved 5/12/26. |
| CGI-028 | No favorites in player picker | Favorites section added to picker modal. Resolved 5/12/26. |
| CGI-029 | No club profile doc or branding foundation | clubs/{clubId}/profile doc defined, populated, useClubProfile hook built, GamesPage brand header added, ClubProfilePage built. Resolved 5/13/26. |
| CGI-030 | Club name hardcoded in ProfilePage | ProfilePage now reads club name from useClubProfile. Resolved 5/13/26. |
| CGI-031 | clubId not explicit in invite links | clubId now embedded as URL param in sendInvite. Resolved 5/13/26. |
| CGI-032 | Inconsistent page header styles across pages | page-title CSS class standardized across all pages. Resolved 5/13/26. |
| CGI-033 | New course form pre-loading existing course data | CourseEditView useEffect now checks courseId before populating. Resolved 5/13/26. |
| CGI-034 | NewGamePage not defaulting to primary course | primaryCourseId added to club profile doc; NewGamePage auto-selects. Resolved 5/13/26. |
| CGI-037 | JoinPage shows password form when invite already claimed | alreadyClaimed state added. Resolved 5/15/26. |
| CGI-038 | claimInvite partial-claim state leaves Auth/Firestore split | claimInvite now recovers from auth/email-already-exists. Resolved 5/15/26. |
| CGI-039 | No self-serve password reset for members | LoginPage: "Forgot password?" inline flow. Resolved 5/15/26. |
| CGI-040 | No admin password reset for members | RosterPage edit panel: "Send Password Reset" button. Resolved 5/15/26. |
| CGI-043 | tnmlTeam not saving on edit | updateMember payload was missing tnmlTeam field. Resolved 5/18/26. |
| CGI-044 | GamesPage landing page bare/uninformative | Redesigned as single scrolling feed. Resolved 5/18/26. |
| CGI-045 | No Info/Help page | InfoPage built at /info. Resolved 5/18/26. |
| CGI-046 | No Venmo support | venmoHandle field added. Resolved 5/18/26. |
| CGI-047 | ProfilePage shows "Profile not found" with no stats | Guard replaced with helpful message. Resolved 5/18/26. |
| CGI-048 | RosterPage "Not Active" tab — should be removed | Tab removed (later restored in Session 23 for deactivated members). Resolved 5/18/26. |
| CGI-049 | No expired token indicator on pending invites | EXPIRED badge added. Resolved 5/18/26. |
| CGI-050 | No User Guide in app | ClubGolf_UserGuide.html created, linked from InfoPage. Resolved 5/18/26. |
| CGI-051 | Portrait score view mislabeled "Scorecard" | Renamed to "Game Points". Resolved 5/18/26. |
| CGI-052 | GHIN lookup not accessible during game setup | GHIN button added below date field in NewGamePage. Resolved 5/18/26. |
| CGI-053 | GameCard share crashes — clipboard API undefined | Fallback added: textarea + execCommand('copy'). Resolved 5/18/26. |
| CGI-054 | Lighthouse accessibility contrast failures | Fixed .gc-crack-badge and .tab-item span. Resolved 5/18/26. |
| CGI-055 | Missing main landmark in index.html | div#root changed to main#root. Resolved 5/18/26. |
| CGI-056 | robots.txt serving index.html | Created public/robots.txt. Resolved 5/18/26. |
| CGI-057 | Tabler Icons CDN blocking render | Preload async pattern applied. Resolved 5/18/26. |
| CGI-058 | "Last Round" label misleading on GamesPage | Renamed to "Last Game". Resolved 5/18/26. |
| CGI-059 | GamesPage header missing member last name | Brand header updated. Resolved 5/18/26. |
| CGI-060 | User Guide scoring rules incorrect | Corrected scoring rules throughout. Resolved 5/18/26. |
| CGI-061 | Invite emails going to spam/blocked | sendInvite CF updated to send from tjunker9@gmail.com. Resolved 5/26/26. |
| CGI-062 | defaultTees not populating for game creator (slot 0) | Race condition fixed in NewGamePage auto-fill effect. Resolved 5/26/26. |
| CGI-063 | Tee selection is free-text input — error-prone | Replaced with select dropdowns in NewGamePage + RosterPage. Resolved 5/26/26. |
| CGI-064 | No way to start on a hole other than hole 1 | startingHole + numHoles added; ScoreEntryPage wraparound sequence. Resolved 5/26/26. |
| CGI-065 | GamesPage stats strip showing 0s after stats migration | Stats strip read path updated from stats.field to stats.all.field. Resolved 5/27/26. |
| CGI-066 | No way to deactivate a member without deleting them | deactivateMember CF added; disables Auth account, sets inviteStatus:'skip'. RosterPage Deactivate button with two-step confirmation. Resolved 5/27/26. |
| CGI-067 | Deactivated members invisible in UI — no way to reactivate | Not Active tab restored to RosterPage (tab order: Active/Pending/To Invite/Not Active). reactivateMember CF added. Reactivate button in edit panel for skip members. Resolved 5/27/26. |
| CGI-068 | No way to edit member role from RosterPage | Role select added to edit panel for accepted members; reads/writes clubs/{clubId}/roles/{uid}. Resolved 5/27/26. |
| CGI-069 | Member emails incorrect from bulk import | updateMemberEmails.cjs script built and run — 62 emails corrected. Resolved 5/29/26. |
| CGI-070 | defaultTees blank for bulk-imported members | setDefaultTees.cjs script built and run — 159 members set to Blue. Resolved 5/29/26. |
| CGI-071 | handicapUpdatedAt null for all bulk-imported members | backfillHandicapUpdatedAt.cjs built and run — 188 members backfilled from createdAt. Resolved 5/29/26. |
| CGI-072 | HI Age shows "Never set" in NewGamePage for all players | handicapUpdatedAt now written in ProfilePage saveHI, useRoster addMember/updateMember, and NewGamePage auto-fill slot 0. Resolved 5/29/26. |
| CGI-073 | ProfilePage saveHI rejecting plus handicaps (< 0) | Validation changed from `parsed < 0` to `parsed < -10`. Resolved 5/29/26. |
| CGI-074 | useRoster addMember defaulting null HI to 0 | addMember now stores null when HI is blank (CGAD-041 compliance). Resolved 5/29/26. |
| CGI-B046 | Stats nested shape not reflected in GamesPage read | Covered by CGI-065. Resolved 5/27/26. |
| CGI-075 | No custom domain — firebase URL not member-friendly | Purchased club-golf.app via GoDaddy. Configured Firebase Hosting custom domain. DNS + SSL cert provisioned 5/31/26. |
| CGI-076 | No in-app notification when a new app version is available | Added useRegisterSW hook in App.jsx — shows "A new version is available / Update Now" banner when SW update is waiting. Resolved 5/31/26. |
| CGI-077 | Invite email uses old firebase URL and lacks install instructions | functions/index.js updated: inviteUrl changed to club-golf.app, logo src updated, iPhone + Android install instructions added to email body. Resolved 5/31/26. |
| CGI-078 | User Guide tabs and pages out of date after Session 23 changes | Updated ClubGolf_UserGuide.html: 5-tab description corrected, Leaderboard page card added, History leaders card reference removed, Profile updated with Admin Panel note, Admin tab card removed, HI Age badge and Starting Hole/numHoles documented. Resolved 5/31/26. |
| CGI-079 | Admin cannot set Venmo handle from RosterPage | venmoHandle field added to edit panel + updateMember payload. Resolved 6/2/26. |
| CGI-080 | No member CSV export for mail merge | exportMembers.cjs exports active_members.csv + pending_members.csv. Resolved 6/2/26. |
| CGI-081 | Game edit access not restricted — any authenticated user could edit any active game | Implemented access gate in ScoreEntryPage.jsx: useEffect checks game.createdBy vs user.uid and isAdmin; non-creators auto-redirect to read-only viewer. Resolved 6/10/26. |
| CGI-082 | Home screen live game card shows no game state | Added getLiveGameStatus() helper to GamesPage.jsx — shows current hole + running score (Hole {n} · A-B · date), derived client-side using same holeSequence/resumeHole logic as ScoreEntryPage. Resolved 6/13/26. |
| CGI-083 | Non-admin viewer (ViewerPage) had no way back to home | Added rp-back-btn to portrait header (navigate to /games). Added optional onBack prop to ScorecardView, rendered as sc-back-btn in sc-left-block when provided; ViewerPage passes onBack for landscape. ScoreEntryPage's ScorecardView usage unaffected. Resolved 6/13/26. |
| CGI-084 | GHIN sync Cloud Function timeout on large roster | Increased syncHandicaps timeout to 180s via onCall options. Resolved 6/18/26. |
| CGI-085 | GHIN sync writing to wrong Firestore path | Profile doc path corrected from `clubs/${clubId}/profile` (3 segments) to `clubs/${clubId}/profile/profile` (4 segments). Resolved 6/18/26. |
| CGI-086 | NewGamePage wizard restructured; GHIN lookup link removed | NewGamePage redesigned as 4-step wizard (Setup/Teams/Settings/Review). GHIN external anchor link removed — per-game sync replaces it. Resolved 6/19/26. |
| CGI-087 | GHIN sync missing non-Tripoli members (e.g. Benjamin Juech) | fetchGhinHandicap cascade now tries Tripoli+WI → WI; single-result null-HI path cascades instead of stopping; parseGhinHI treats "NH" as null; guest state picker added to GHIN modal. Resolved 6/19/26. |
| CGI-088 | Guest player name not editable in Setup step | ng-pr-name renders as inline text input for isGuest slots. Resolved 6/19/26. |
| CGI-089 | Guest tees not defaulting to creator's default tees | handleMemberSelect guest branch now sets teeSet from creatorDefaultTees ref. Resolved 6/19/26. |
| CGI-090 | GHIN sync showing NaN for players with "NH" handicap index | parseGhinHI helper added — treats "NH" and any non-numeric HI value as null. Resolved 6/19/26. |
| CGI-091 | Multiple guests showing duplicate names in updated list | allUpdates now uses updates array only (guestUpdates is a subset — was double-counted). Resolved 6/19/26. |
| CGI-092 | Second guest silently dropped from GHIN sync | Guest catch block now pushes to ambiguousEntries with empty candidates so client surfaces "could not look up" message. Resolved 6/19/26. |
| CGI-093 | NewGamePage Review step showed wrong team grouping | Review step Teams filter read the stale `s.team` slot property (set once at slot creation, never updated) instead of `teamForSlot(s.slotId)`, which reads the live `teamAssignment` state. Actual saved games were unaffected — `handleCreate`/`handleUpdate` already used `teamForSlot`. Fixed Review step filter to match. Resolved 6/30/26. |

---

## Issue ID Sequence
Next open issue: **CGI-094**
Next bug: **CGI-B047**
Next deferred item: **CGI-D025**
