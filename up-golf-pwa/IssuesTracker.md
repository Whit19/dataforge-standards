# IssuesTracker.md — UP Golf PWA
**Statuses:** `Open` | `In Progress` | `Resolved` | `Deferred` | `Won't Fix`

---

## Open Issues

| ID | Description | Status |
|----|-------------|--------|

---

## Resolved Issues

| ID | Description | Resolution |
|----|-------------|-----------|
| ISS-001 | Backend platform selection | Firebase chosen ✅ |
| ISS-002 | Frontend framework selection | React + Vite ✅ |
| ISS-003 | Hosting platform | Firebase Hosting + GoDaddy ✅ |
| ISS-004 | Push notifications on iOS | Working on iOS 16.4+ ✅ |
| ISS-005 | Player count | 32 players ✅ |
| ISS-006 | R3 Tiebreaker "Roulette" | Deferred — admin manual override |
| ISS-007 | Score entry permissions | Foursome-based R2/R4 ✅ |
| ISS-008 | Photo upload storage | Phase 4 ✅ |
| ISS-009 | Offline conflict resolution | Last-write-wins ✅ |
| ISS-011 | Bulk CSV player import | Admin SDK Cloud Function ✅ |
| ISS-012 | OutingSetupTab outing ID mismatch | Confirmed outing_2026 ✅ |
| ISS-013 | Admin auto-login during import | Admin SDK server-side ✅ |
| ISS-014 | Score entry number picker style | Confirmed working — closed ✅ |
| ISS-015 | Magic strings scattered | constants.js + firestorePaths.js ✅ |
| ISS-020–062 | Various scoring, skins, payment bugs | All resolved ✅ |
| ISS-036 | onRoundLocked Cloud Function disabled (Eventarc Service Agent IAM permissions blocked 2nd-gen trigger deploy) | Resolved 2026-03-30: IAM role confirmed present, function uncommented in functions/index.js and deployed ✅ |
| ISS-046 | functions/prizes.js used Firestore subcollection paths instead of top-level collections | Resolved: all paths corrected to top-level (matches, scores, payments, etc.) ✅ |
| ISS-053 | resolvedMatches (client-side winner derivation) used Combined Gross scoring for R4 instead of Best Ball | Resolved: per-hole Math.min(teammates) applied for R4 ✅ — see ISS-141/ISS-145 below for the full removal of this client-side recompute pattern |
| ISS-056 | R4 match winner showed stale/wrong value when read from Firestore | Resolved at the time by always recomputing R4 winner client-side rather than trusting the stored value — this workaround became the template for AdminPayments.jsx's broader live-recompute architecture, which is the direct root of ISS-141 below. Superseded 2026-06-30: Firestore is now the trusted source again, with the underlying write-path bug actually fixed in functions/prizes.js |
| ISS-063 | window.confirm() blocked | Inline confirmation UI ✅ |
| ISS-064 | enableIndexedDbPersistence() deprecated | Replaced with persistentLocalCache API ✅ |
| ISS-065 | apple-mobile-web-app-capable deprecated | Updated + added mobile-web-app-capable ✅ |
| ISS-066 | Fixed headers gap behind status bar | `.app` background + `top:0` ✅ |
| ISS-067 | Chat screen residual layout issues | Phase 4.5 ✅ |
| ISS-068 | SW scope conflict blocks iOS push | injectManifest + iife ✅ |
| ISS-069 | PushPermissionPrompt debug panel | Removed ✅ |
| ISS-070 | VAPID key typos | Fixed via copy/paste ✅ |
| ISS-071 | Auto-subscribe effect causes prompt flash | Removed auto-subscribe ✅ |
| ISS-072 | Push notifications fire twice on iOS | Fixed: foreground guard in onBackgroundMessage ✅ |
| ISS-073 | window.confirm() in AdminCourseLibrary | Replaced with inline "Sure? Yes/No" UI ✅ |
| ISS-074 | Foreground notifications silently dropped | Fixed: onMessage calls swRegistration.showNotification() ✅ |
| ISS-075 | Fixed header content cut-off | Fixed: ResizeObserver using el.offsetHeight ✅ |
| ISS-076 | Push SW SyntaxError + iOS delivery | Superseded by ISS-087 ✅ |
| ISS-077 | Tee times render as flat string | Fixed: split on newline/comma, render as pills ✅ |
| ISS-078 | TournamentSummaryCard shows all dashes | Fixed: field name mismatches ✅ |
| ISS-079 | Overall leaderboard +/- all dashes | Fixed: r1Par/r2Par/r4Par added ✅ |
| ISS-080 | Live +/- shows wrong value | Fixed: calculated against played holes par only ✅ |
| ISS-081 | R3 scramble prizes never calculated | Fixed: scrambleResults exposes playerIds + holes ✅ |
| ISS-082 | AdminPayments crashes to white screen | Fixed: scrambleWinners useMemo ordering ✅ |
| ISS-083 | R3 tee times empty in Groups + Home | Fixed: generatePairings assigns tee times ✅ |
| ISS-084 | TournamentSummaryCard wrong age group winners | Fixed: normalised strings 'U50'/'50+' ✅ |
| ISS-085 | Groups icon too small / wrong in nav | Fixed: tight viewBox SVG ✅ |
| ISS-086 | Match cards show "Down X" | Fixed: always show "Up X" / "All Square" ✅ |
| ISS-087 | Foursomes posted notification not displayed in foreground | Fixed: 2s delay + token wipeout bug fixed ✅ |
| ISS-088 | FCM token wiped when PC session opened | Fixed: checkAndRefreshToken never writes fcmTokens:[] ✅ |
| ISS-089 | Double notifications on iOS PWA | Fixed: onBackgroundMessage foreground guard ✅ |
| ISS-090 | Accessibility: user-scalable=no blocks pinch zoom | Fixed: viewport meta → maximum-scale=5.0 ✅ |
| ISS-091 | Accessibility: no main landmark | Fixed: div#root → main#root in index.html ✅ |
| ISS-092 | Accessibility: login text contrast failure | Fixed: login-screen__sub/footer → rgba(255,255,255,0.75) ✅ |
| ISS-093 | Scores nav icon wrong proportion | Fixed: new landscape leaderboard SVG in App.jsx ✅ |
| ISS-094 | Leaderboard header icon outdated | Fixed: matched to new nav icon in LeaderboardScreen.jsx ✅ |
| ISS-095 | App spins forever on offline cold-start | Fixed: setLoading(false) unblocked from await loadProfile() in useAuth.jsx ✅ |
| ISS-096 | ScoreEntry shows "Online · Auto-saved" when offline | Fixed: useSyncStatus isOnline drives sync row text ✅ |
| ISS-097 | Enable Notifications prompt flashes on Home nav | Fixed: isSupportChecked flag gates isInitialising in usePushNotifications.js ✅ |
| ISS-098 | index.html served stale after deploy | Fixed: no-cache header added to firebase.json ✅ |
| ISS-099 | Stale FCM tokens — Mike + Tom stopped receiving pushes | Fixed: dead token pruned by FCM multicast; re-subscribed via site data clear + new Enable bell button ✅ |
| ISS-100 | No in-app way to re-enable notifications | Fixed: bell badge/button added to Dashboard welcome card ✅ |
| ISS-099b | birdieNotify.js had no dead token pruning | Fixed: pruning + per-token warn logging added ✅ |
| ISS-099c | push.js dead token pruning too narrow | Fixed: now prunes all FCM failures ✅ |
| ISS-101 | FCM token accumulation — stale tokens pile up on same device | Fixed: fcmTokens array now replaced (not appended) on every subscribe and token refresh ✅ |
| ISS-102 | Green bell static — no tap handler | Fixed: converted to button with onClick → handleReEnableNotifications ✅ |
| ISS-103 | Dashboard shows stale cached status on reopen | Fixed: outing onSnapshot uses includeMetadataChanges:true ✅ |
| ISS-104 | Leaderboard sorted by raw gross, not score-to-par | Fixed: cumulativeLeaderboard sorts by _totalTpar; round tabs re-sorted by tpar ✅ |
| ISS-105 | Age filter only applied to Overall tab | Fixed: ageGroup added to round rows; prepRoundRows() filters all round tabs ✅ |
| ISS-106 | Medals awarded by stale hook rank | Fixed: RoundTab medals use loop index (displayRank) ✅ |
| ISS-107 | Score-to-par colors inverted | Fixed: swapped colors in RoundTab and OverallTab ✅ |
| ISS-108 | Admin cannot enter scores for other foursomes | Fixed: isAdmin bypass + group picker in ScoreEntry.jsx ✅ |
| ISS-109 | Player usernames inconsistent format | Fixed: all 32 accounts migrated to FirstnameLastname format ✅ |
| ISS-110 | Danny Sanicki had Firestore doc but no Auth account | Fixed: orphan doc deleted; re-added via Admin Panel ✅ |
| ISS-111 | Auth accounts for deleted/unknown players still present | Fixed: migrate-users.js deleted all non-CSV accounts ✅ |
| ISS-112 | AdminPayments showed stale entry fee paid status | Investigated: stale offline cache on dormant PC tab. No code change needed ✅ |
| ISS-113 | Duplicate push notifications on iOS and Android | Fixed: native push event in SW; compat SDK removed; onMessage removed ✅ |
| ISS-114 | Foursomes save creates duplicate/stale docs — missing G prefix | Fixed: firestorePaths.js foursome ref uses `_G${groupNumber}` ✅ |
| ISS-115 | Match doc ID missing M prefix | Fixed: firestorePaths.js match ref uses `_M${matchNumber}` ✅ |
| ISS-116 | AdminFoursomes save didn't update match docs | Fixed: Team 1/VS/Team 2 UI; saves both foursome + match docs in batch ✅ |
| ISS-117 | Skins opt-in cutoff — isClosed = round?.isLocked; hasScores prop removed; verified dry run 2026-06-04 | Fixed ✅ |
| ISS-118 | displayName flashes email address on app load | Fixed: useAuth.jsx displayName fallback is '' not user.email ✅ |
| ISS-119 | Test data tool R3 delete missed foursomes collection docs | Fixed: R3 branch now also deletes foursomes docs ✅ |
| ISS-120 | R3 Leaderboard sort wrong when tiebreaker applies | Fixed: r3Leaderboard derived from scrambleResults (full tiebreaker sort); rank uses s.place ✅ |
| ISS-121 | Leaderboard defaults to Overall instead of active round | Fixed: LeaderboardScreen reads outing status on load, sets tab to current round ✅ |
| ISS-122 | MoneyScreen defaults to wrong round | Fixed: uses last locked round among 2/3/4, default 2 if none locked ✅ |
| ISS-123 | Tiebreaker scores not shown in Leaderboard or MoneyScreen | Fixed: Back 9 Score/Last 3 Holes Score shown in R3Tab, RoundTab, and ScrambleResultsList ✅ |
| ISS-124 | Scramble team name entry by players didn't work | Fixed: TeamNameEditor removed; team names set by admin only in AdminFoursomes ✅ |
| ISS-125 | Dashboard Round 4 shows wrong game format | Fixed: STATUS_CONFIG text replaced with dynamic FORMAT_LABELS[activeRound.format] lookup ✅ |
| ISS-126 | No way for players to confirm round is complete | Fixed: "Complete Round" button + confirmation panel in ScoreEntry; writes completedAt to foursome doc ✅ |
| ISS-127 | Dashboard shows tee time card after round complete | Fixed: myFoursomeComplete flag shows Round Complete card when completedAt set or round locked ✅ |
| ISS-128 | MoneyScreen winnings shown before round locked | Fixed: WinningsTab gated on rounds[activeRound].isLocked; placeholder shown when unlocked ✅ |
| ISS-129 | Dashboard My Winnings shows live mid-round winnings | Fixed: myWinnings useMemo gates each component on corresponding round lock ✅ |
| ISS-130 | Inconsistent fonts across screens | Fixed: leaderboard number cells use var(--font-ui); admin mono strings normalised; redundant fontFamily:inherit removed ✅ |
| ISS-131 | Chat doesn't scroll to bottom on load | Fixed: ChatThread.jsx scroll-to-bottom with setTimeout(0) + image load handlers ✅ |
| ISS-132 | Foursomes R2/R4 cards missing holes-thru indicator | Fixed: "Thru X" added to match cards, computed as max thru across all four players ✅ |
| ISS-133 | Round Complete showing prematurely before round locked | Fixed: split into three player states; myFoursomeComplete now isLocked only; myScoresSubmitted dual condition (DEC-133) ✅ |
| ISS-134 | bulkCreatePlayers FieldValue not imported — overwrites enrolledPlayerIds | Fixed: FieldValue imported from firebase-admin/firestore; arrayUnion used correctly ✅ |
| ISS-135 | ageCategory field on some player docs causing age filter miss | Fixed: Charlie Pietz patched (ageGroup added, ageCategory removed); Mike Dethloff ageCategory removed; all 32 docs confirmed clean ✅ |
| ISS-136 | "Add Dining Event" Save button unreachable on iOS PWA — modal/background scroll broken | Fixed: AnnouncementModal/DiningEventModal converted to inline forms (AnnouncementForm/DiningEventForm) in InfoScreen.jsx, matching NotesTab pattern. No overlay used ✅ (DEC-136) |
| ISS-137 | ProfileEditModal Save/Cancel hidden after first save | Fixed: modal restructured as flex column with scrollable fields + sticky footer ✅ |
| ISS-138 | ProfileEditModal Save/Cancel hidden behind bottom nav on iPhone | Fixed: modal overlay z-index raised to 2000; footer padding-bottom set to calc(env(safe-area-inset-bottom) + 72px) ✅ |
| ISS-139 | firstYear field missing from all 32 player docs | Fixed: set-first-years.js bulk script run via Admin SDK; all 32 docs updated ✅ |
| ISS-140 | Players saw empty Skins tab on Money screen instead of live standings — only admins saw results | Fixed: payments Firestore read rule opened to all signed-in players (was admin/own-doc only), which had been silently blocking the broad paymentsForOuting query useSkins.js needs for all non-admins ✅ (DEC-139) |
| ISS-141 | outing_2026 archive missing purse, location, game format, and all player winnings | Resolved: AdminArchiveTournament.jsx read payments.prizeWon/skinsWon (fields that don't exist — real field is winningsEarned), outing.location/startDate/endDate (fields that don't exist on the outings doc), and rounds.gameFormat (real field is format). All three field-name mismatches fixed; location/dates now derived from rounds[].courseName/date ✅ (DEC-155) |
| ISS-142 | functions/prizes.js onRoundLocked silently wrote zero match results despite logging success | Resolved: calculateAndWriteMatchResults and calculateAndWriteScrambleResults now count actual writes and throw if matches/scramble data existed but nothing was updated, instead of silently committing an empty batch. prizesCalculatedAt (the idempotency guard) is no longer written unless real results were produced, so a future failure can retry on the next lock event instead of being permanently skipped ✅ (DEC-156) |
| ISS-143 | R3 scramble prizes paid wrong tier to 2nd-place team (3rd-place amount) | Resolved: place-assignment tie-detection in calculateAndWriteScrambleResults compared totalGross only, ignoring the backNine/lastThree tiebreakers the sort above it already used correctly — this collapsed 3 distinct-ranked teams with equal totalGross into one shared place. Fixed to compare the full tiebreaker chain ✅ (DEC-157) |
| ISS-144 | functions/prizes.js recalculateAllPlayerWinnings hardcoded match count as 8 instead of deriving from actual data | Resolved: per-round match count (R2/R4 separately) now derived from matchesSnap, matching the live AdminPayments.jsx pattern, with || 8 fallback retained only for the zero-matches-exist edge case ✅ (DEC-156) |
| ISS-145 | AdminPayments.jsx and useSkins.js recomputed match winners/prizes/skins live in the browser instead of reading persisted Firestore data | Resolved: this was a deliberate fallback built in March 2026 while onRoundLocked was disabled (ISS-036) and never reverted after the fix deployed 2026-03-30 — the two systems silently diverged for 3 months with no indication anything was wrong, since the live recompute always showed correct numbers even though Firestore never did. CollectPaySection now reads matches.winner/prizePerPlayer and payments.scrambleWinnings directly; useSkins.js's stored-result-takes-precedence merge logic was already correct, only its comments were updated ✅ (DEC-158) |
| ISS-146 | TournamentResultsPDF.jsx ignores the :year URL param, always shows ACTIVE_OUTING_ID's live data | Partially resolved: new useHistoricalResults.js hook + branching logic added so /history/:year now reads from history_ collections when the requested year is not the active outing. Tested against /history/2025. Known remaining gaps: per-round leaderboard detail tables and hole-by-hole skins grids are NOT populated in historical mode (render empty — no historical equivalent built yet); ageGroup is not present in history_standings so the Grp column shows '—' for archived years. See Deferred section ✅ (DEC-159, DEC-160) |
| ISS-147 | TournamentResultsPDF.jsx cover header/print bar/footer hardcoded ACTIVE_OUTING_YEAR even in historical mode | Resolved: derived displayYear constant now used in all three locations, correctly showing the viewed year instead of always the active year ✅ |

---

### ISS-117 — Skins opt-in cutoff
**Status:** Resolved
**Description:** Skins opt-in was closing too early when scores were present 
rather than waiting for round lock.
**Resolution:** `isClosed = round?.isLocked` — round lock is the only cutoff. 
`hasScores` prop removed from `SkinsOptInCard`. Verified in dry run 2026-06-04.

### ISS-118 — In-app PDF guides
**Status:** Resolved
**Description:** Players needed access to App Setup and User Guide documentation 
from within the app.
**Resolution:** Two PDFs uploaded to Firebase Storage (`Guides/` folder). Two 
cards added to Info screen under a "GUIDES" section label. Tap opens PDF in 
device viewer. Deployed 2026-06-04.

### ISS-136 — Add Dining Event Save button unreachable (iOS PWA)
**Status:** Resolved
**Description:** On the "Add Dining Event" form (InfoScreen), the Save/Cancel 
buttons below the Notes field were unreachable on iOS PWA standalone. The modal 
content extended past the visible viewport behind the fixed bottom nav, and 
neither the modal nor the background page would scroll vertically to reveal it 
(horizontal scroll worked, vertical did not).
**Resolution:** Converted `AnnouncementModal` and `DiningEventModal` (both used 
the shared `.modal-overlay`/`.modal` floating overlay pattern) to inline forms 
(`AnnouncementForm`, `DiningEventForm`) shown in-page when `showForm` is true, 
matching the existing `NotesTab` pattern. No floating overlay remains in 
`InfoScreen.jsx`. See DEC-136 for full investigation/attempts.

### ISS-140 — Players couldn't see live Skins standings on Money screen
**Status:** Resolved
**Description:** Discovered after the R1 dry run in production: the Money >
Skins tab showed "Skins results will appear as scores are entered" for every
player, every round, regardless of actual score data, while admins saw the full
live pot/standings/hole-by-hole board on the same screen. Winnings tab was
unaffected.
**Resolution:** Root cause was `firestore.rules` — the `payments` read rule
(`allow read: if isAdmin() || request.auth.uid == resource.data.playerId`) only
permits reading one's own payment doc. `useSkins.js` queries ALL payment docs at
once (`queries.paymentsForOuting`) to build the skins opt-in roster; Firestore
rejects any such broad query for a non-admin under a per-document-owner rule, so
the listener silently failed and `payments` stayed `[]` client-side for every
player. Changed read rule to `allow read: if isSignedIn()` in both the
`outings/{outingId}/payments` and top-level `payments` blocks. Write access
unchanged. See DEC-139.

---

## Deferred

| ID | Description | Deferred To |
|----|-------------|-------------|
| ISS-006 | R3 roulette tiebreaker | Phase 6 admin override |
| ISS-010 | Emoji picker | Phase 4 polish |
| ISS-021 | playerInit() duplicated | Phase 7 cleanup |
| ISS-023 | TechnicalArchitecture subcollection mismatch | Phase 7 cleanup |
| ISS-148 | TournamentResultsPDF historical mode: per-round leaderboards + hole-by-hole skins grids not populated | Phase 8B — needs historical equivalent of r1Leaderboard/r2Leaderboard/etc. and skins hole detail, not just cumulative standings |
| ISS-149 | Original cause of ISS-142's silent empty-batch match-result writes never fully root-caused | Unresolved — fix prevents recurrence (loud failure going forward) but the historical mechanism for why the original 2026 R2/R4 batches wrote nothing despite no thrown error was never determined. Logged for awareness only, not blocking |
