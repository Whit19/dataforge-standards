# TNPL — Session Starter

**Project:** Thursday Night Paddle League (TNPL) PWA
**Status:** Planning
**Last updated:** 2026-09-22

## One-line description
A PWA for Thursday Night Paddle League: live scoring, member sign-up, Elo rankings, and weekly match creation/viewing — similar to the UP Golf and Club Golf PWAs.

## Current status
Project kickoff just completed. No code written yet. Repo, Notion project row, and doc scaffolding are being set up in this session.

## Next priorities
1. [PLACEHOLDER — fill in after repo/Claude Project are created] Review the existing TNPL context docs (Excel Elo workbook, current/future-state notes, email+form screenshots) and turn them into a formal requirements list.
2. [PLACEHOLDER] Decide on data model for players, matches, weeks, and Elo history in Firestore.
3. [PLACEHOLDER] Decide on availability-collection flow (replacing the manual Mailmeteor + Google Form process).
4. [PLACEHOLDER] Decide on the match-pairing algorithm (ELO-diff constraint, currently ≤75 in the manual process).

## Key decisions so far
See DecisionLog.md. No technical decisions logged yet beyond stack selection (see TP-001).

## Known context (gathered, not yet decisions)
The following is background pulled from Tom's existing TNPL planning docs and Excel workbook — it describes the CURRENT manual process and an existing Elo scoring system, not final app decisions. See TechnicalArchitecture.md for details.

## Open questions
- [PLACEHOLDER] Confirm GitHub org/username to use for the repo (assumed `Whit19`, matching UP Golf / AFAS convention).
- [PLACEHOLDER] Confirm whether TNPL should reuse Tom's existing Firebase account/plan or get its own project within it (currently assumed: same Firebase account, new Firebase project).
