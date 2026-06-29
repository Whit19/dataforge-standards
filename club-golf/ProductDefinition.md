# Club Golf — Product Definition Document
**Version 1.0 | May 2026**

---

## 1. North Star

> **Club Golf is the player's app — not the pro's app.**

Every feature decision, every screen, every interaction is evaluated against one question:
*Does this make the experience better for the golfer, or does it serve the administrator?*

If it's primarily for the administrator, it belongs on the "future" or "no" list.

---

## 2. What Club Golf Is

Club Golf is a **mobile-first PWA** for golfers who want to play money games and track their personal history across any format, at any club, with any group.

It is:
- A **game day companion** — players see their tee time, group, format, and live scores in one place
- A **personal record** — every game a logged-in player participates in is saved to their history
- A **format library** — a growing set of configurable game formats with side game options
- A **social scorecard** — live hole-by-hole scoring with instant feedback, visible across foursomes
- A **multi-club platform** — each club gets its own branded instance ("Club Golf — Tripoli") with its own roster, games, and history

It is **not**:
- A pro shop management tool
- A tee sheet or booking system
- A GHIN/handicap integration platform (handicaps are manually entered)
- A multi-round tournament engine (v1)
- A payment processing platform (who paid is tracked, payments happen offline)
- A push notification system (v1)
- A chat/messaging platform (v1)

---

## 3. Brand & Identity

**Platform name:** Club Golf
**Club instances:** Club Golf — [Club Name] (e.g., "Club Golf — Tripoli")
**Tagline (working):** *Your game. Your history. Your club.*

The platform name "Club Golf" is intentional — it's a play on "golf club," signaling that this belongs to the players, not the institution. Each club gets their own named instance with their own branding, roster, and game history — but they all run on the same platform.

---

## 4. The Two Users

### 4A — The Player (Primary)
A golfer who belongs to a club, plays regularly, and wants:
- To spin up or join a game easily
- To see live scores during the round
- To know where they stand without digging through menus
- To look back at their history ("I've played Vegas 12 times, won 8")
- A fun, interactive experience — not a data entry form

**Guest players** are supported but limited: they participate in a game, appear on the scorecard, but have no account, no history, and no login. They are identified by name only. Guests do not count toward any billing metric.

### 4B — The Game Organizer (Trusted Player)
Any registered player can be designated as a game organizer by a club admin. The organizer is still a player first — they just have additional abilities:
- Create and configure games
- Add foursomes to an existing game
- Add tee times and optional payment status per player
- Lock a completed round
- Discard a game before it is locked (no stats saved)

There is no "pro shop admin" role in the player-facing app. Club-level admin (managing the roster, setting up the club instance) is handled separately and is not part of the daily player experience.

---

## 5. The Game Lifecycle

Every game in Club Golf follows the same three-state model, borrowed from UP Golf:

```
SCHEDULED → SCORING → LOCKED
```

**SCHEDULED**
- Game has been created and configured
- Players can see their group, tee time, format, and side game opt-ins
- No scores entered yet
- Payment status visible (paid / not paid — informational only)
- Organizer can still add foursomes or edit settings

**SCORING**
- Round is live
- Score entry open to players in their own group
- Live leaderboard visible to all players in the game
- Organizer can view and correct scores
- Organizer can add late foursomes (they join mid-round)

**LOCKED**
- Round is complete — organizer locks it
- Final results calculated and displayed
- If organizer chooses "Save," all scores are written to player history and stats
- If organizer chooses "Discard," the game is deleted — no stats saved
- Locked games cannot be edited

---

## 6. Core Features

### 6A — Game Day Card
When a player opens Club Golf on the day of a game, they see a single card at the top of their home screen:
- Game name and format
- Their tee time
- Their group members (with handicaps if applicable)
- Side games they've opted into
- Payment status (theirs)
- State indicator: Scheduled / Scoring / Locked
- One-tap "Enter Scores" button when in Scoring state

No navigation required to get to their game. This is the player's front door.

### 6B — EZ Games
Any registered player can create a quick game without an organizer role:
- Pick a format from the library
- Add 2–4 players (from roster or as guests)
- Set handicaps if applicable
- Choose side games (optional)
- Share a join link or code

**Cross-Foursome Play:** If another player from the same club creates or joins a game using the same game code, their foursome's scores are added to the same leaderboard. Players in different foursomes compete against each other on a shared live leaderboard. This is a signature feature.

### 6C — Score Entry & Groups Page
This is the most important screen in the app. Design principles:
- **One thumb operation** — all score entry reachable with a single thumb
- **Your group first** — hole-by-hole scores for your foursome displayed prominently
- **Always know where you stand** — current points/net score/skins status visible without leaving the screen
- **Instant feedback** — birdie indicator, skins carry display, points flash on save
- **Field view** — one tap to expand to full leaderboard across all foursomes
- Mirrors the UP Golf Groups + ScoreEntry pattern but refined for multi-format and multi-foursome play

### 6D — Live Leaderboard
- Updates in real time as scores are entered across all foursomes
- Shows standings by primary format result (net score, points, etc.)
- Side game status shown separately (current skins leader, Nassau status, etc.)
- Filterable: your group only vs. full field
- Works in both portrait and landscape

### 6E — Player History & Stats
Per registered player, per game format:
- Games played, won, lost, tied
- Point totals, differentials, averages
- Best round, worst round
- Birdies, proxes, and other bonus events
- Timeline view: every saved game with date, course, result
- This is a primary differentiator from GG — players own their own record

### 6F — Club Roster & Handicaps
- Club admin manages the roster
- Handicap index entered manually per player (no GHIN integration in v1)
- Players can update their own handicap index from their profile
- Course handicap calculated automatically from slope/rating at game creation

---

## 7. Game Format Library

Formats are offered in a defined library. Each format has configurable options (like 6-Point's crack, birdie mode, casket settings). Players are not building scoring engines — they are choosing from options within a known format.

### 7A — Primary Formats (v1 Build Order)

| # | Format | Notes |
|---|--------|-------|
| 1 | 6-Point Game | Already built — port from Club Golf |
| 2 | Stroke Play (Individual) | Gross or net, with optional max score per hole |
| 3 | Stableford | Configurable point values per score type |
| 4 | Best Ball (2-person teams) | Net or gross; best score of the pair counts per hole |
| 5 | Scramble | 2 or 4 person; configurable tee requirement (e.g. must use each player's tee at least 2x) |
| 6 | Match Play | 1v1 or 2v2; gross or net; configurable concession rules |
| 7 | Nassau | Front 9, Back 9, Overall; configurable press rules |
| 8 | Skins | Gross or net; carryover on/off; configurable tie rules |
| 9 | Vegas | 2v2; configurable scoring (combine digits or multiply) |
| 10 | 3-2-3 / Variable Best Ball | Configurable: how many scores count per hole by par (e.g. 3 on par 3, 2 on par 4, 3 on par 5) |
| 11 | Bingo Bango Bongo | Configurable point values for each of the 3 scoring events |
| 12 | Wolf | Player rotation; configurable point structure |

### 7B — Side Games (Opt-in at Game Creation)

Side games run alongside any primary format. Players opt in individually or as a group at game creation. Side game results are tracked and displayed separately from the primary format.

| Side Game | Notes |
|-----------|-------|
| Skins | Can be added to any primary format; gross or net |
| Closest to Pin | Par 3s only; tracks proximity result per hole |
| Longest Drive | Designated holes; tracked manually |
| Nassau | Can be a side bet on any stroke play format |
| Sandy / Pully / Manual Bonus | Configurable bonus events (ported from 6-Point) |

### 7C — Configurable Options (Apply Across Formats)
- Gross vs. Net vs. % of Net (e.g. 80% of handicap)
- Max score per hole (triple bogey cap, double bogey cap, or none)
- Mulligan rules (none, 1 per round, 1 per 9)
- Concession rules (gimme distance or none)
- Tee set per player (each player can play different tees)
- Handicap strokes shown on scorecard (on/off)

### 7D — Format Combination Rule
One primary format + any number of opt-in side games. Two primary formats running simultaneously are not supported (v1). This keeps the scoring engine clean and the player experience understandable.

---

## 8. The "Adding Foursomes" Flow

This is how Club Golf handles groups larger than 4 without turning it into a tournament engine:

1. Game creator sets up the game with their foursome (4 players)
2. Game is in SCHEDULED state
3. Creator or any designated organizer adds additional foursomes:
   - Pick players from club roster or add guests
   - Assign tee time per foursome
   - Mark payment status per player (paid / not paid — informational)
4. Each added player sees the game on their home screen Game Day Card
5. When the round goes SCORING, all foursomes enter scores independently
6. All foursomes compete on the same live leaderboard
7. Organizer locks the round when all foursomes are complete

This model supports everything from a casual 4-person game to a 10-foursome club event — without building a tournament engine.

---

## 9. Multi-Club Architecture

Each club on the platform is a distinct instance:
- Own roster
- Own game history
- Own branding (club name in header)
- Own admin player(s)
- Shared platform infrastructure (auth, scoring engine, format library)

Players belong to one primary club but can be invited as guests to games at other clubs. Cross-club events (like district matches) are a future phase — v1 focuses on within-club play.

**Data isolation:** All game and player data is scoped under `clubs/{clubId}` — consistent with existing Club Golf architecture.

---

## 10. Monetization Model

### Recommended: Hybrid Freemium + Club Subscription

**Free tier — single club, core features:**
- Up to X players on roster (TBD — suggest 20)
- All game formats
- Full player history
- EZ Games
- No time limit

**Paid tier — "Club Golf Pro" (~$50–$75/month per club):**
- Unlimited roster
- Multi-organizer support (more than 1 trusted player per club)
- Advanced stats and history export
- Priority support
- Future: cross-club events, district match format

**Per-event option (for non-club organizers):**
- District coordinators or one-off event organizers can pay per event (~$10–$25/event)
- No club subscription required
- Players participate as guests or create individual accounts
- Targets the "runs 2 events a year" organizer that a club subscription doesn't fit

**Guest players:** Always free, never counted in billing.

---

## 11. What This Is NOT (v1 Hard No List)

| Feature | Status | Notes |
|---------|--------|-------|
| GHIN / handicap API integration | ❌ No | Manual entry only |
| Tee sheet / booking | ❌ No | Out of scope permanently |
| In-app payment processing | ❌ No | Who paid is tracked; money moves offline |
| Push notifications | ❌ No | Deferred — UP Golf FCM was a significant build |
| Chat / messaging | ❌ No | Deferred — consider emoji reactions on leaderboard as lightweight alternative |
| Multi-round tournament engine | ❌ No (v1) | Season standings (aggregated history) yes; multi-round single event no |
| Two simultaneous primary formats | ❌ No | One primary + side games only |
| Public/open game joining (no invite) | ❌ No | Invite/code only for access control |
| Video / swing analysis | ❌ No | Out of scope permanently |
| Custom scoring engine builder | ❌ No | Configurable options within defined formats only |

---

## 12. Relationship to Existing Club Golf Codebase

This platform IS Club Golf — it is not a separate product. The current Club Golf app (Phase 1G, 36.5 hours invested) is the foundation. The path forward:

**Immediately reusable:**
- 6-Point scoring engine (`sixPointEngine.js`, `crackEngine.js`)
- Handicap calculation (`handicapCalc.js`)
- Multi-tenant Firestore architecture (`clubs/{clubId}/...`)
- Auth model (Firebase Auth, invite flow)
- UI component library (design system, BottomTabBar, GameCard, ScorecardView)
- Course data model (tee sets, hole stroke index)
- Member invite flow (sendInvite / claimInvite / deleteMember Cloud Functions)

**Needs extension:**
- Game creation wizard — new formats added to existing step-wise flow
- Score entry — multi-format engine, multi-foursome live scoring
- Leaderboard — format-aware, cross-foursome, side game display
- Player profile / history — extended per-format stats
- Game lifecycle state machine (Scheduled → Scoring → Locked — port from UP Golf)
- Organizer role (trusted player designation per club)

**New builds:**
- EZ Games flow
- Game Day Card on home screen
- Format library + configurable options engine
- Side games layer
- Vegas, 3-2-3, Wolf, Bingo Bango Bongo scoring engines
- Multi-club subscription/billing layer (Phase 5)

---

## 13. Success Metrics (v1)

- A player can create and complete a game in under 3 minutes of setup time
- Score entry for one hole takes under 10 seconds per player
- A new player joining for the first time can enter scores without any instructions
- Player history is visible and meaningful after 5 saved games
- The app works on poor course WiFi / cell coverage (offline score entry with sync)

---

## Version History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | May 2026 | Initial definition — pre-architecture |
