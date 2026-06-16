# Master Claude Protocol — DataForge
**Apply this to every Claude Project and every session. No exceptions.**
Last updated: 2026-06-03

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

| Project | Stack | Notion Page |
|---------|-------|-------------|
| UP Golf PWA | React + Vite + Firebase | [link] |
| Club Golf | React + Vite + Firebase | [link] |
| AFAS | Azure Functions + Python + Azure SQL + Power BI | [link] |
| HQ Dashboard | Google Apps Script + Claude API + Google Sheets | [link] |
| MicroSynergies | Node.js + Express + Anthropic API | [link] |

---

## 3. Session Startup — Required Every Session

### 3a. Standard repo structure
Every project GitHub repo must have a `/docs` folder at the root containing all MD files:

```
[project-repo]/
├── docs/
│   ├── SessionStarter.md
│   ├── TechnicalArchitecture.md
│   ├── ProjectRoadmap.md
│   ├── DecisionLog.md
│   ├── IssuesTracker.md
│   ├── BestMethods.md
│   └── TimeLog.md
├── src/
├── .gitignore
└── ...
```

### 3b. Files to upload at session start
MD files live in `[project-repo]/docs/` on GitHub — pull the latest versions and upload them at the start of each session. Do not store MD files permanently in the Claude Project.

Note that ALL Client level-reference MDs (non-code) line in OneDrive under C:\Users\tjunk\OneDrive\Documents\_DATAFORGE\_CLIENTS\[ClientName]/ rather than in a GitHub repo. 

**Upload ALL of the following at session start — do not start coding without them:**

| File | Purpose |
|------|---------|
| `SessionStarter.md` | Current status, next priorities, key decisions |
| `TechnicalArchitecture.md` | Data models, stack, file structure |
| `ProjectRoadmap.md` | Phased task list with status |
| `DecisionLog.md` | All decisions with rationale — prevents relitigating |
| `IssuesTracker.md` | Open, deferred, resolved issues |
| `BestMethods.md` | Hard-won lessons — read before writing any code |
| `TimeLog.md` | Session time tracking |

**Never start writing code until all required files are uploaded and read.**

### 3c. Session startup prompt template
```
New session for [Project Name].
Uploading docs from /docs folder now.
[upload files]
Today's focus: [1-2 sentence goal]
```

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
1. Claude generates a CC prompt for each changed MD file (one prompt per file)
2. Tom pastes CC prompts into Claude Code in VS Code — files written to disk
3. Tom saves updated files back to the Claude Project (no downloads needed)

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

**Prefixes by project:**
| Project | Prefix |
|---------|--------|
| UP Golf | DEC |
| Club Golf | CGAD |
| AFAS | AFAS |
| HQ Dashboard | AD / DD / PD |
| MicroSynergies | MS |

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

Run this every time a new project starts:

- [ ] Create GitHub repo (private) — create `/docs` folder, MD files live here
- [ ] Create all 7 standard MD files locally in `/docs` and commit to GitHub
- [ ] Create Claude Project in claude.ai — load MD files from GitHub at session start (do not store permanently in Claude Project)
- [ ] Add project row to Notion Projects database
- [ ] Add services to Notion Services database (linked to project)
- [ ] Add `.env` / `local.settings.json` to `.gitignore` before first commit
- [ ] Confirm no secrets in any committed file
- [ ] Add local folder path and OneDrive folder path to Notion project row
- [ ] Set Status to `In Progress`

---

## 14. Session End Checklist

**Only update MD files that actually changed this session — not all 7 every time.**
**Do not generate any MD file updates until explicitly told: "Update the docs."**

At session end, Tom will say "Update the docs" — only then generate updates. For each file that changed:

1. Claude generates a CC prompt (one per file) to write the updated content to `docs/[filename].md`
2. Tom pastes CC prompts into Claude Code in VS Code — files written to disk
3. Tom saves updated files back to the Claude Project — no downloads needed
4. Commit and push — GitHub is the source of truth

**Files to consider per session (only update if changed):**
- `SessionStarter.md` — almost always needs updating (status, completed items, next priorities)
- `ProjectRoadmap.md` — update if phases or tasks changed
- `DecisionLog.md` — append only if new decisions were made
- `IssuesTracker.md` — update if issues were opened, deferred, or resolved
- `TechnicalArchitecture.md` — update only if stack or data models changed
- `BestMethods.md` — update only if new lessons were learned
- `TimeLog.md` — always update (duration + one-sentence summary)

**Also at session end:**
- [ ] Update `Last Updated` date in Notion project row

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
<type>: <short summary line>

<bullet: what changed>
<bullet: what changed>
<bullet: why or outcome>


**Types:** `feat` (new feature), `fix` (bug fix), `chore` (config/docs/tooling), 
`refactor` (code change, no behavior change)

### When to generate
Claude generates both automatically when the session end checklist runs — 
i.e. when Tom says "Update the docs." No separate prompt needed.

### Also update TimeLog.md
The TimeLog entry summary line should match the session summary in tone and 
brevity — one sentence capturing the session's main accomplishment.

---

## 15. What Claude Should Never Do

- **Never update, generate, or modify any MD files unless explicitly told "Update the docs"**
- **Never make any code changes unless explicitly asked — no proactive fixes, no "while I'm here" edits**
- Never write sensitive values (keys, tokens, passwords) into any file or chat response
- Never read sensitive values from screenshots
- Never start writing code before all session docs are uploaded
- Never make multiple speculative code changes at once when debugging
- Never inline Firestore collection/doc refs outside `firestorePaths.js`
- Never use `&&` chaining in PowerShell commands
- Never use `npm install` without `--legacy-peer-deps` on PWA projects
- Never duplicate information across MD files — reference, don't repeat
- Never re-litigate a decision that exists in the DecisionLog with an Active status
