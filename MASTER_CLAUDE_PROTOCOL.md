# Master Claude Protocol — DataForge
**Apply this to every Claude Project and every session. No exceptions.**
Last updated: 2026-08-20

---

## 1. Who I Am

**Tom Junker** — DataForge LLC
Milwaukee, WI — freelance developer / AI consultant
414-406-6605 · tjunker@dataforge.llc

**Dev environment:** Windows 11, VS Code, PowerShell, Dell Tower Plus (Intel Ultra 7-265, 64GB RAM, RTX 5060)
**Laptop:** Windows 11, VS Code (secondary)
**Primary mobile:** iPhone

---

## 2. Active Projects

**One repo = one docs folder = one DecisionLog prefix, always.** Client relationships are tracked only in Notion (a Client field/relation on the Projects database) — never encoded in repo names, `dataforge-standards` subfolder names, or DecisionLog prefixes. A single client can have multiple, unrelated projects; each still gets its own row here and its own row in Notion. See Section 13a for what to do when a new project starts under an existing client.

| Project | Client (if applicable) | Stack | Notion Page |
|---------|------------------------|-------|-------------|
| UP Golf PWA | — | React + Vite + Firebase | [link] |
| Club Golf | — | React + Vite + Firebase | [link] |
| AFAS | — | Azure Functions + Python + Azure SQL + Power BI | [link] |
| Kids HQ | — | Google Apps Script + Claude API + Google Sheets | [link] |

*(MicroSynergies sub-projects — `ask-the-docs`, `tech-team-wiki`, `knowledge-hub` — are currently inactive. Add rows here individually if/when work resumes, following Section 13a — do not add a single combined "MicroSynergies" row.)*

---

## 3. Session Startup — Required Every Session

### 3a. Doc storage — dataforge-standards repo
All project documentation lives in the **public** `dataforge-standards` GitHub repo
(github.com/Whit19/dataforge-standards), not in the private code repos.
This allows Claude to fetch docs directly via raw GitHub URLs with no authentication.

```
dataforge-standards/                    ← public repo
├── MASTER_CLAUDE_PROTOCOL.md           ← root; fetched by all projects
├── NEW_PROJECT_KICKOFF.md              ← root; used before a project folder exists
├── up-golf-pwa/                        ← one subfolder per project
│   ├── SessionStarter.md
│   ├── TechnicalArchitecture.md
│   ├── ProjectRoadmap.md
│   ├── DecisionLog.md
│   ├── IssuesTracker.md
│   ├── BestMethods.md
│   └── TimeLog.md
├── club-golf/
│   └── ...
├── afas/
│   └── ...
└── Kids_HQ/
    └── ...
```

Private code repos (e.g. `up-golf-pwa`) contain only source code — no docs.
Client-level reference MDs (non-code) stay in OneDrive under
`C:\Users\tjunk\OneDrive\Documents\_DATAFORGE\_CLIENTS\[ClientName]\`.
These are uploaded manually when needed; they are not fetched via URL.

### 3b. Fetching docs at session start
Claude fetches all docs at the start of every session using `curl` via the
bash tool, using the raw GitHub URLs configured in each Claude Project's
Instructions field. No manual uploads are needed or expected.

**Do not use `web_fetch` for this.** `raw.githubusercontent.com` is served
through a CDN that can silently return a stale cached copy of a file for a
period after a fresh commit — no error, no 404, just old content. This was
confirmed directly: `web_fetch` returned a doc with a stale "Last updated"
date immediately after a commit GitHub's own UI already showed as current.
Cache-busting query params do not fix this — GitHub's CDN ignores
unrecognized query params on this endpoint. A direct `curl` request from
the sandbox bypasses the CDN layer that `web_fetch` goes through and
reliably returns the live file, so it's the standard method — not a
fallback reached for only when a fetch looks suspicious.

This fetch mechanic is duplicated directly in each Claude Project's
Instructions field (not just referenced here), because Instructions are
guaranteed to be in context at session start — nothing needs to be fetched
first for Claude to see them. If this section and a Project's Instructions
field ever disagree, update both together.

**Raw URL pattern:**
`https://raw.githubusercontent.com/Whit19/dataforge-standards/main/[project]/[file].md`

**Fetch command:**
```bash
curl -s "https://raw.githubusercontent.com/Whit19/dataforge-standards/main/[project]/[file].md"
```

**Files Claude fetches for every session:**

| File | Purpose |
|------|---------|
| `MASTER_CLAUDE_PROTOCOL.md` | Always fetched first — overrides everything |
| `SessionStarter.md` | Current status, next priorities, key decisions |
| `TechnicalArchitecture.md` | Data models, stack, file structure |
| `ProjectRoadmap.md` | Phased task list with status |
| `DecisionLog.md` | All decisions with rationale — prevents relitigating |
| `IssuesTracker.md` | Open, deferred, resolved issues |
| `BestMethods.md` | Hard-won lessons — read before writing any code |
| `TimeLog.md` | Session time tracking |

**Never start working until all URLs have been fetched and read.**
If a fetch fails (network error, 404), Claude must stop and tell Tom before
proceeding.

The Project Instructions field in each Claude Project contains the full
list of URLs for that project. Update the URLs there if a file is renamed
or moved.

### 3c. Session startup prompt template
Claude fetches all docs automatically from the URLs in Project Instructions.
Tom only needs to state the session focus:

```
Today's focus: [1-2 sentence goal]
```

Claude responds with: "[Project] docs loaded. [One sentence current status from
SessionStarter]. Ready."

---

## 4. How Code Is Delivered — Claude Code (CC) Prompts

**All code changes are delivered as Claude Code prompts for VS Code — not inline in chat.**

### Why
- Fewer tokens consumed in chat
- Changes apply directly to the correct files
- No copy/paste errors

### Format for a CC prompt
Each prompt must be:
1. **Self-contained** — includes all context CC needs (file path, what to change, why)
2. **Single responsibility** — one logical change per prompt
3. **File-specific** — always names the exact file(s) to modify

### CC prompt template
```
File: src/[path/to/file.jsx]

Context: [1-2 sentences about what this file does and why we're changing it]

Change: [Precise description of what to add/modify/remove]

Rules:
- [Any project-specific rules that apply, e.g. "All Firestore refs through firestorePaths.js"]
- [e.g. "No && chaining in PowerShell"]
- [e.g. "Always --legacy-peer-deps"]
```

### One file per prompt
Never combine full-file rewrites or edits to multiple files into a single CC
prompt — even for closely related changes like an end-of-session doc sync
touching several MD files. Large multi-file prompts risk truncation in the
chat window before they can be copied. Generate one prompt per file, even if
it means pasting several prompts in sequence.

### Nested code fences in CC prompts
If a CC prompt's content itself contains a fenced code block (e.g. an MD file
rewrite that includes its own ``` SQL/code block, run-order references, etc.),
wrap the entire CC prompt in four backticks (````) instead of three. Matching
fence depth between the outer prompt and an inner block causes the inner
closing fence to prematurely close the outer one — splitting the prompt into
multiple disconnected chat windows and breaking copy-paste.

### CC prompts are delivered as downloadable MD files
Every CC prompt — whether for a code file, a new component, or an end-of-session
doc update — must be delivered as a downloadable `.md` file, not as inline chat text.

**Why:**
- MD files can be opened and copied in full without scrolling or truncation risk
- Files persist in the session for reference if Claude Code needs to be re-run
- Naming convention makes it clear which file each prompt targets

**Naming convention:** `CC_[ShortDescription].md`
Examples: `CC_TournamentResultsPDF.md`, `CC_AppJsx_ResultsRoute.md`, `CC_SessionStarter.md`

**One MD file per CC prompt** — same rule as one prompt per file. Never combine
multiple CC prompts into a single MD file.

**How:** Claude uses the `create_file` tool to write each CC prompt to
`/mnt/user-data/outputs/CC_[ShortDescription].md`, then presents all CC prompt
files together at the end using `present_files`. Tom downloads and pastes into
Claude Code in VS Code.

**This applies to all projects, all sessions, without exception.**

---

### When to use chat vs CC
| Use chat (Claude.ai) | Use Claude Code (VS Code) |
|---------------------|--------------------------|
| Design decisions | All code changes |
| Data model discussions | Bug fixes |
| Architecture planning | File creation |
| Reviewing MD files | Refactors |
| Generating CC prompts | Running terminal commands |

---

## 5. MD File Protocols

### 5a. Standard MD files — every project must have all of these
- `SessionStarter.md` — updated every session (most critical)
- `TechnicalArchitecture.md` — updated when stack/models change
- `ProjectRoadmap.md` — updated as phases complete
- `DecisionLog.md` — append-only; never delete entries
- `IssuesTracker.md` — move issues between Open/Deferred/Resolved
- `BestMethods.md` — add lessons as they are learned
- `TimeLog.md` — duration + one-sentence summary per session

### 5b. MD file update workflow (end of every session)
1. Claude generates a CC prompt MD file for each changed doc (one per file)
2. Tom downloads the CC prompt MD files and pastes into Claude Code in VS Code — files written to disk
3. Tom commits and pushes to `dataforge-standards` repo — GitHub is the source of truth
4. Claude fetches the latest version automatically at the next session start via URL

No manual re-upload to Claude Project needed. GitHub commit = live for next session.

**Do not rely on screenshots to capture updated MD content — always get the text.**

### 5c. Rules for writing MD files
- **No duplicate information** — if a fact lives in TechnicalArchitecture, do not repeat it in SessionStarter. Reference it instead.
- **No sensitive data in MD files** — no API keys, tokens, passwords, connection strings. Use `[see local.settings.json]` or `[env var: VARIABLE_NAME]` as placeholders.
- **No screenshots for critical info** — tokens, API keys, emails, IDs must be copy/pasted as text, never captured in images.
- **Keep SessionStarter lean** — it is loaded every session and consumes tokens. Link to other docs for detail.
- **DecisionLog is append-only** — never edit or delete past entries; mark superseded decisions as `Superseded by [ID]`.

---

## 6. Security Rules — Non-Negotiable

- **Never put API keys, tokens, passwords, or connection strings in MD files or chat**
- **Never read sensitive values from screenshots** — always require copy/paste as text
- **Environment variables only** for secrets: `local.settings.json` (Azure), `.env` (Node.js), Firebase App Settings
- **`.env` and `local.settings.json` are always in `.gitignore`** — confirm before first commit on any project
- **`serviceAccountKey.json` is never committed** — delete after use
- When referencing a key/token in docs, use the variable name only: e.g. `ANTHROPIC_API_KEY` — never the value

---

## 7. Windows / PowerShell Rules

These apply to every project. No exceptions.

| Rule | Detail |
|------|--------|
| No `&&` chaining | PowerShell does not support `&&`. Run commands sequentially or use `;` |
| Always `--legacy-peer-deps` | Required for all projects using vite-plugin-pwa. Never plain `npm install` |
| Keep projects outside OneDrive | `node_modules` + OneDrive = corruption and build errors |
| `npm run build` before `firebase deploy` | Always build first; never deploy a stale build |
| Functions deploy separately | `firebase deploy --only functions` — not bundled with hosting deploy |

---

## 8. Firebase / Firestore Rules

| Rule | Detail |
|------|--------|
| All Firestore refs through `firestorePaths.js` | Never inline collection/doc refs in components |
| All constants in `constants.js` | Never hardcode collection names, status strings, or format labels in JSX |
| `persistentLocalCache` API | `enableIndexedDbPersistence()` is deprecated — do not use |
| Offline cold-start | `setLoading(false)` must NOT be inside `await loadProfile()` — app spins forever if Firestore is offline |
| Cloud Function public access | Must be set manually in Cloud Run console after every new function deploy |
| `functions/.env` never committed | Always in `.gitignore` |
| Every new Firebase project | Separate Firebase project per app — never share a Firebase project across apps |

---

## 9. PWA Rules

| Rule | Detail |
|------|--------|
| `injectManifest` + `rollupFormat: 'iife'` | Only SW strategy that works with Firebase + iOS |
| `index.html` → `no-cache` | Add to `firebase.json` |
| SW files → `no-cache` | Same — stale SW = invisible updates |
| `*.js` / `*.css` → `immutable` | Hashed filenames are safe to cache forever |
| `window.confirm()` blocked in PWA standalone | Always use inline confirmation UI |
| `user-scalable=no` is accessibility violation | Use `maximum-scale=5.0` instead |
| VAPID key: copy/paste only | Never retype — one wrong character = push silently fails |
| PWA icons: white background | 180px (Apple touch), 192px, 512px |

---

## 9a. Daily Summaries → Notion

Projects that generate a recurring daily/periodic summary (briefings, digests, reports) should push each run's output to the shared **Daily Summaries** Notion database rather than any per-project or manual storage (e.g. OneNote). This is a workspace-wide database, not per-project — one Notion internal integration, connected once, writes to it from every project's backend.

**Schema:** `Name` (title), `Date`, `Category` (select — project-specific values, e.g. Kids HQ uses Sports/School), `Top Priority` (text), `Project` (relation to the Projects database). Full content (the actual summary body — cards, sections, whatever the project's summary consists of) goes in the page body via Notion's page-content API, not a database property.

**Setup per project** (see the Dashboard's New Project Checklist for the full sequence): confirm the shared integration is connected to the database (one-time, workspace-wide — only needed once ever, not per project), store the integration secret in that project's own secrets store (Script Properties for Apps Script projects, `.env`/equivalent for others — never hardcoded), and wire the project's summary-generation code to POST to Notion's API on each run, tagging the `Project` relation to that project's row in the Projects database.

**Upsert behavior:** re-running a summary for a date/category that already has an entry should update that existing Notion page, not create a duplicate — check for an existing entry (by Date + Category + Project) before creating.

Kids HQ is the reference implementation of this pattern (see its DecisionLog for the specific integration details).

---

## 10. Debugging Protocol

**Do not jump to conclusions. Look for the most obvious solution first.**

1. Read the error message fully before forming a hypothesis
2. Check the most obvious cause first (typo, wrong import path, missing file)
3. Upload the relevant source files before asking Claude to diagnose logic bugs — do not ask Claude to guess
4. State what you have already tried before asking for help
5. One fix at a time — do not stack multiple speculative changes

---

## 11. Decision Log Protocol

Every architectural or product decision gets logged. Format:

```markdown
### [PROJECT-PREFIX]-[NNN] — [Short title]
**Decision:** What was decided
**Rationale:** Why
**Alternatives considered:** What else was on the table
**Status:** Active | Superseded by [ID]
```

**Prefixes by project (not by client — see Section 13a):**
| Project | Prefix |
|---------|--------|
| UP Golf | DEC |
| Club Golf | CGAD |
| AFAS | AFAS |
| HQ Dashboard | AD / DD / PD |

*When a new project starts under an existing client, it gets its own prefix — never reuses another project's prefix, even under the same client. E.g. if MicroSynergies resumes: `ask-the-docs` → ATD, `tech-team-wiki` → TTW, `knowledge-hub` → KH.*

**Always check the DecisionLog for the next available ID before adding an entry.**

---

## 12. Issues Tracker Protocol

Every bug or deferred item gets logged. Format:

```markdown
### [PREFIX]-[NNN] — [Short title]
**Status:** Open | Deferred | Resolved
**Description:** What the issue is
**Resolution:** (fill in when resolved)
```

**Never silently fix an issue without logging it — even small ones that affect behavior.**

---

## 13. New Project Checklist

Run this every time a new project starts. Use `NEW_PROJECT_KICKOFF.md`
(root of `dataforge-standards`, alongside this file) as the actual prompt
to paste into a new Claude.ai chat — it walks through every item below
interactively and generates the MD files and Project Instructions block
for you.

- [ ] Create GitHub repo (private) — source code only, no docs folder needed
- [ ] Create project subfolder in `dataforge-standards` repo — add all 7 standard MD files and commit
- [ ] Create Claude Project in claude.ai — paste raw GitHub URLs for all 7 docs + master protocol into Project Instructions field
- [ ] Confirm `.gitignore` covers `.env`, `local.settings.json`, `node_modules` — do NOT gitignore the whole `.claude` folder if the project uses any project-level Claude Code Skills (see Section 13b); those should stay committed
- [ ] Add project-level Claude Code Skills if applicable (e.g. `pwa-firebase-rules` for React/Vite/Firebase projects) — see Section 13b

**Handled automatically via the Notion connector (Claude.ai chat only — see Section 17):**
Claude creates the Notion Projects database row, sets Status to Planning/In Progress,
adds local folder + OneDrive folder paths, adds the GitHub repo URL, and links
Services rows, as part of running the kickoff prompt. Confirm the row looks right
in Notion rather than re-entering it by hand.

### 13a. Multi-project clients

Client relationships never get encoded into repo names, `dataforge-standards`
subfolder names, or DecisionLog prefixes. A client with multiple projects
(e.g. MicroSynergies with `ask-the-docs`, `tech-team-wiki`, `knowledge-hub`)
gets one repo + one docs subfolder + one prefix *per project*, exactly as if
each were an unrelated client. The only place "this project belongs to this
client" is recorded is a Client field/relation on the Notion Projects database
row. Never create a single combined docs folder or repo for "the client" as
a whole.

### 13b. Claude Code Skills at project setup

Personal skills (`cc-prompt-delivery`, `powershell-windows-rules`,
`session-docs-protocol`, living in `~/.claude/skills/`) apply automatically
to every new project — nothing to do. Project-specific skills
(e.g. `pwa-firebase-rules`) need to be copied into `.claude/skills/` inside
the new repo and committed, so they're available on any machine, not just
the one where they were first created.

---

## 14. Session End Checklist

**Only update MD files that actually changed this session — not all 7 every time.**
**Do not generate any MD file updates until explicitly told: "Update the docs."**

At session end, Tom will say "Update the docs" — only then generate updates. For each file that changed:

1. Claude generates a CC prompt MD file (one per file) targeting `[project]/[filename].md` in the `dataforge-standards` repo
2. Tom downloads the CC prompt MD files and pastes into Claude Code in VS Code — files written to disk
3. Tom commits and pushes `dataforge-standards` — GitHub is the source of truth
4. No Claude Project re-upload needed — Claude fetches latest via URL at next session start
5. **Sync Notion.** Update the `Last Updated` date property on the project's Notion page (Projects data source, page ID `37462bde-ae68-80a2-ab5a-c2f6fcbae3c6` for Kids HQ) to the session's date. Use the `notion-update-page` tool with `update_properties`, setting `date:Last Updated:start` to today's date and `date:Last Updated:is_datetime` to `0`. This happens every time docs are updated at session end — not just when something notable changed — so the Notion row always reflects the most recent session date.

**Files to consider per session (only update if changed):**
- `SessionStarter.md` — almost always needs updating (status, completed items, next priorities)
- `ProjectRoadmap.md` — update if phases or tasks changed
- `DecisionLog.md` — append only if new decisions were made
- `IssuesTracker.md` — update if issues were opened, deferred, or resolved
- `TechnicalArchitecture.md` — update only if stack or data models changed
- `BestMethods.md` — update only if new lessons were learned
- `TimeLog.md` — always update (duration + one-sentence summary)

**Also at session end:**
Claude updates the `Last Updated` date on the Notion project row automatically
via the Notion connector, as part of the same "Update the docs" turn — see
Section 17. No separate manual step, but this only fires in Claude.ai chat;
see Section 17 for what to do if a session happened entirely in Claude Code.

---

## 15. What Claude Should Never Do

- **Never update, generate, or modify any MD files unless explicitly told "Update the docs"**
- **Never make any code changes unless explicitly asked — no proactive fixes, no "while I'm here" edits**
- Never write sensitive values (keys, tokens, passwords) into any file or chat response
- Never read sensitive values from screenshots
- Never start working before all session docs have been fetched via `curl` (not `web_fetch`)
- Never make multiple speculative code changes at once when debugging
- Never inline Firestore collection/doc refs outside `firestorePaths.js`
- Never use `&&` chaining in PowerShell commands
- Never use `npm install` without `--legacy-peer-deps` on PWA projects
- Never duplicate information across MD files — reference, don't repeat
- Never re-litigate a decision that exists in the DecisionLog with an Active status
- Never encode a client relationship into a repo name, docs folder, or DecisionLog prefix — see Section 13a

---

## 16. Session Summary & GitHub Commit Message

At the end of every session, Claude generates both of the following without
being asked — as part of the standard session close:

### Session Summary
A 3–5 sentence plain-English summary of what was accomplished. Covers:
- What was built or fixed
- Any significant debugging or environment issues resolved
- Current status of the project

### GitHub Commit Message
A conventional commit format message ready to paste into GitHub:
```
<type>: <short summary line>

<bullet: what changed>
<bullet: what changed>
<bullet: why or outcome>
```

**Types:** `feat` (new feature), `fix` (bug fix), `chore` (config/docs/tooling),
`refactor` (code change, no behavior change)

### When to generate
Claude generates both automatically when the session end checklist runs —
i.e. when Tom says "Update the docs." No separate prompt needed.

### Also update TimeLog.md
The TimeLog entry summary line should match the session summary in tone and
brevity — one sentence capturing the session's main accomplishment.

---

## 17. Notion Automation Scope

Notion is connected via the Notion MCP connector, **in Claude.ai chat only.**
Claude Code has no access to it — connecting a service in claude.ai does not
extend to Claude Code, which would need its own separate MCP server setup to
reach Notion at all. Do not assume a Claude Code session can read or write
Notion just because chat can.

**Fully automatic, no confirmation needed, whenever these happen in chat:**
- **Kickoff** (Section 13): create the Projects database row, set Status,
  add local/OneDrive folder paths, add the GitHub repo URL, link Services rows.
- **"Update the docs"** (Section 14): update `Last Updated` on the project row,
  and update Status if the session moved the project to a new phase
  (e.g. Planning → In Progress, or In Progress → Complete).

**Before the first automatic write in a new area of Notion**, Claude confirms
the actual database/property names by reading the schema via the connector
rather than guessing — property names (e.g. "Local Folder" vs "Local Path")
must match exactly or the write will silently create a new property instead
of filling the existing one.

**If a session happens entirely in Claude Code** (no round-trip through chat),
the Notion row does not get updated automatically — there's no connector
there to do it. Either do a short check-in in chat afterward ("update the
docs" still triggers the Notion sync even if the code work already happened
in Claude Code), or update the Notion row by hand for that session.
