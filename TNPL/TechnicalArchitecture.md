# TNPL — Technical Architecture

**Last updated:** 2026-09-22

## Stack
- Frontend: React + Vite
- Backend / data: Firebase (Firestore assumed for data, Firebase Auth assumed for member sign-up, Firebase Hosting assumed for deployment — [PLACEHOLDER: confirm each]
- Repo: https://github.com/Whit19/TNPL (private)
- Local folder: `C:\Dev_Projects\TNPL`

## Data model
[PLACEHOLDER — not yet designed. Will need at minimum: Players, Weeks/Matches, Match Groups, Sets/Scores, Elo History, Availability Responses.]

## File structure
[PLACEHOLDER — standard Vite/React layout assumed until decided otherwise.]

## Reference material gathered at kickoff
Tom provided existing planning docs for TNPL that describe both a manual current-state process and an already-built Excel Elo system. These are reference material for design, not yet architectural decisions.

### Existing Elo rating system (from League Elo Rating System — Season 3 Setup & Workflow, and TNPL_MAIN.xlsx)
- K-factor: 32
- Starting Elo by flight (Season 2): F1 = 1575, F2 = 1525, F3 = 1400. Season 3 uses each player's prior-season "Next Season Start" value instead of flight defaults.
- Team Elo = average of the 2 players' individual Elos
- Expected score (Team A) = 1 / (1 + 10^((TeamB_Elo − TeamA_Elo) / 400))
- Margin multiplier = 0.6 + 0.16 × (|point_diff| − 1)
- Per-set adjustment = K × margin_multiplier × (actual_result − expected_score), same adjustment applied to both teammates
- End-of-week update: all 3 sets use start-of-week Elo as basis; the 3 set adjustments are summed and applied once per week
- Next-season regression: shrinkage = 0.7 × sets_played / (sets_played + 20); next_start = 1500 + shrinkage × (final_elo − 1500)
- Workbook (TNPL_MAIN.xlsx) currently tracks this in Excel: Wk# sheets (source of truth for weekly scores), Player List, Master Data v2 (long-format consolidation), Season Rankings, Season History.

### Current manual weekly process (from Current/Future State notes and Emails_Forms)
1. Every Monday, an availability email goes out via Mailmeteor (mail-merge from Google Sheets) with a link to a personalized Google Form ("Can you play this week?", "If needed, can you play 2 matches?", preferred/blocked times, notes).
2. Tom reviews responses and follows up with non-responders.
3. Tom manually creates matches from responses, pairing players of similar ability — constraint used: Elo difference between highest- and lowest-ranked player in a match ≤ 75 (adjustable).
4. Pairings are emailed out.
5. Requested changes are handled manually (player requests a change, Tom approves/disapproves).
6. Matches are printed, players fill in results on paper, Tom transfers results to Excel, and Elo is recalculated.

### Future-state goals stated by Tom
- PWA similar to UP Golf / Club Golf, reusing much of the same player base as Club Golf
- Member sign-up with contact info
- Automated availability emails
- Live scoring and season tracking
- Automated Elo scoring with an editable K-factor and other variables

## Known constraints / preferences
- Local dev on Windows 11, VS Code, PowerShell.
- [PLACEHOLDER: any Firebase project ID / naming convention, e.g. `tnpl-pwa` similar to `up-golf-pwa`]
