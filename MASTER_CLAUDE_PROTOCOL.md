# Master Claude Protocol — DataForge
**Apply this to every Claude Project and every session. No exceptions.**
Last updated: 2026-09-05 (restructured — see 3d "Read the Docs" [new explicit command], Section 4 [Path A/B fork for CC prompts vs. direct Claude Code edits, corrected to reflect chat-drafts-code-changes as the common case], Section 14 [consolidated "Update the Docs" procedure, absorbing former 14/16/17, with an explicit Case 1/2 fork mirroring Section 4 and a formal chat→Claude Code→chat handoff loop for the Session Summary])

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
├── AFAS/
│   └── ...
└── Kids_HQ/
    └── ...
```

**Casing matters — these are literal folder names in a raw GitHub URL.**
`AFAS/` is capitalized (confirmed live, repeatedly, via direct fetch — a
lowercase `afas/` 404s). If a project's Claude Project Instructions field
disagrees with the actual repo casing, trust the repo and fix the
Instructions field, not the other way around.

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

A minimal version of this fetch mechanic is duplicated directly in each
Claude Project's Instructions field (not just referenced here), because
Instructions are guaranteed to be in context at session start — nothing
needs to be fetched first for Claude to see them. This is a genuine
bootstrapping problem, not a stylistic choice: this file's own instruction
to fetch itself can't be read until *something* has already told Claude to
fetch it.

**Keep the duplicated copy minimal — bootstrap only, not policy.** The
Instructions field should contain only the URLs, the fetch command, the
"curl not web_fetch" rule, and a line stating that once
`MASTER_CLAUDE_PROTOCOL.md` is loaded, it is authoritative and wins any
disagreement with the Instructions-field snippet. It should NOT contain the
rationale, the CDN-caching explanation, or any paraphrased behavioral
policy (like session-start procedure detail, or the "Read the docs"
command in 3d) — that content only needs to exist once, here, since it's
only ever relevant *after* this file has already been fetched. A smaller
duplicated surface has less to go stale, and when this file changes
(as it did adding 3d), only the bootstrap snippet itself needs
re-syncing — which will rarely change, since it's just URLs and a fetch
command, not behavior.

**Self-check at session start.** After fetching `MASTER_CLAUDE_PROTOCOL.md`,
compare the live fetch-table/URL list here against what the Instructions
field's bootstrap snippet actually says. If they disagree (a renamed file,
a changed URL, a stale file list), tell Tom and offer the corrected
snippet to paste in — don't silently proceed on whichever version happened
to load first. This is what makes "update both together" (the rule this
replaced) actually enforceable instead of relying on Tom to remember every
time this file changes.

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

The Project Instructions field in each Claude Project contains the minimal
bootstrap version of this (URLs + fetch command + "curl not web_fetch" +
"this file wins on disagreement") — not the full prose of this section.
Update the URLs there if a file is renamed or moved.

### 3c. Session startup prompt template
Claude fetches all docs automatically from the URLs in Project Instructions.
Tom only needs to state the session focus:

```
Today's focus: [1-2 sentence goal]
```

Claude responds with: "[Project] docs loaded. [One sentence current status from
SessionStarter]. Ready."

### 3d. "Read the Docs" — the explicit, on-demand command

This is a distinct trigger from the automatic session-start fetch in 3b/3c,
and gets its own explicit definition here because it was previously
undefined — a real source of Claude missing it.

**When Tom says "Read the docs" (or "reload the docs," "refresh the docs" —
same trigger) at any point in a session** — not just at the start — Claude
must:

1. Re-run the full `curl` fetch (Section 3b) for every file in the fetch
   table, right then, even if this session already loaded them earlier.
   Treat anything already in context as potentially stale — that's the
   entire point of the command; don't skip a file because "I already have
   it."
2. Read the freshly-fetched content before saying anything else or taking
   any other action. If a file changed since the session started (e.g.
   another Claude Code session updated it), the new content wins —
   silently update working assumptions, don't flag the diff unless asked.
3. Confirm briefly: "[Project] docs re-read. [One sentence current status],
   [note anything that changed since this session started, or 'no changes
   since session start']."

**This works identically whether the session is Claude.ai chat or Claude
Code** — both have `curl`/fetch access to the public repo. A Claude Code
session with a local clone may use `git pull` + direct file reads instead
of `curl`, as long as it actually reads live content rather than trusting
what's already in context.

**Do not treat "Read the docs" as a no-op just because docs were already
loaded this session.** The whole reason Tom says it explicitly, mid-session,
is that something may have changed since — a stale answer here defeats the
entire purpose of the command.

---

## 4. How Code Is Delivered — Claude Code (CC) Prompts

*Scope note: this section governs changes to project source code (private repos). End-of-session updates to the 7 `dataforge-standards` MD docs follow their own workflow — see Section 14.*

### 4a. Which path applies — check this first

There are two genuinely different situations here, and the rest of this
section only makes sense once the right one is picked:

**Path A — Tom is typing directly into an already-open Claude Code session,
describing the change in plain language (no prepared prompt).** Whatever
that session already knows from the conversation is all the context it
needs — it already has the repo open. **Just make the edit.** Read the
live file first, edit it with the normal file tools, and — only if Tom has
separately asked for a commit — commit following Section 5d's branch
discipline. **No `CC_*.md` prompt file, no `create_file`/`present_files`
step, nothing to paste anywhere.** Producing an intermediate prompt file
for yourself to then "paste into yourself" is pure overhead with no safety
benefit; the two-step dance in Path B exists specifically to carry context
across a gap between two different sessions, and Path A doesn't have that
gap to bridge.

**Path B — the design/analysis happened in a Claude.ai chat conversation,
and the change needs to reach a Claude Code session that was not part of
that conversation** (a separate session, opened later, with none of that
chat's history). Since that session doesn't have the context, chat has to
write it all down as a self-contained prompt — that's the rest of Section
4 (format, template, delivery mechanics). Chat drafts the prompt, delivers
it as a file, Tom pastes it into Claude Code.

**In practice, for Tom, Path B is the more common path for anything code-
related** — the normal habit is design/discussion in Claude.ai chat first,
then bringing the resulting CC prompt to a Claude Code session to execute.
Path A is what's happening in the narrower case of an already-open Claude
Code session being given a direct, undocumented instruction (as in most of
an ongoing debugging or doc-sync session) — not the default assumption for
"a code change is needed."

**If unsure which path applies, ask one question: does the session that
will make the edit already have everything it needs from its own
conversation, or does it need a written-down spec because it wasn't part
of the conversation where this was decided?** The former is Path A — don't
manufacture a prompt file to hand to yourself. The latter is Path B.

### 4b. Path B — drafting a CC prompt in chat, for a separate Claude Code session

### Why (Path B only)
- Fewer tokens consumed in chat
- Changes apply directly to the correct files once pasted
- No copy/paste errors from inline chat text

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

**This applies whenever Path B (Section 4a) is the right path — i.e. chat
is drafting for a separate Claude Code session to consume later.** It does
not apply to Path A (Claude Code editing directly) — see 4a before assuming
this section governs the current session.

---

### 4c. When to use chat vs Claude Code

This table is about where *design/analysis* work happens, not a claim that
Claude Code can't touch MD files or that chat is the only place to review
them. Note the asymmetry with Section 14: for **doc updates**, a Claude
Code session with a local clone writing directly *is* the default path
(5b). For **code changes**, there's no equivalent single default — which
path applies depends on where the design work happened (4a), and in
practice that's chat more often than not.

| Use chat (Claude.ai) | Use Claude Code (VS Code) |
|---------------------|--------------------------|
| Design decisions | Executing a change — either from a Path B prompt just pasted in, or Path A when the change is decided in the same live conversation |
| Data model discussions | Bug fixes |
| Architecture planning | File creation |
| Drafting a CC prompt to hand off to Claude Code (Path B — the common case for code) | Reviewing/editing MD files directly (default path, Section 14) |
| — | Running terminal commands |

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

### 5b. MD file update mechanics — updated 2026-08-27

*This section covers only the mechanics of **how** a doc update gets written to disk. For **when** this runs, **what else** happens alongside it (Session Summary, commit message, Notion), and the full ordered procedure, see Section 14 — this section is one step inside that procedure, not a standalone trigger.*

**Default path — Claude Code session with a local `dataforge-standards` clone:**
1. Tom gives Claude Code a plain-language summary of what happened this session — facts only. No assumed line numbers, no assumed "current text reads X" snippets, no pre-assigned AD/PD/ISSUE/BM numbers. Can be typed directly into Claude Code, or pasted in from a Claude.ai chat conversation that did the actual design/analysis work — see Section 14c for the formal version of this handoff (the Session Summary file).
2. Claude Code reads each affected file's **current, live content** directly — via its own file tools against the local clone — before writing anything to it.
3. Claude Code determines the next-available AD/PD/ISSUE/BM number itself, by checking the live file at write time — **but first confirm the project actually numbers that file's entries at all.** The `grep -oE "^### (AD|PD|DD|BM|ISSUE)-[0-9]+" *.md | sort -t- -k2 -n -u | tail -20` pattern assumes per-entry `PREFIX-NNN` headers; not every project's files follow it (e.g. AFAS's DecisionLog logs decisions in session-grouped tables with no per-decision ID, and AFAS's BestMethods uses plain titles with no BM-NNN at all). When a project deviates, match that project's own established convention rather than forcing this template onto it — don't invent numbering a project has never used. Numbers (where they exist) are never pre-assigned by Tom in the summary — see the rationale below for why.
4. Claude Code writes the changes directly to disk. No intermediate `CC_*.md` prompt file is generated for docs updates — same reasoning as Path A in Section 4a: this session already has the file open, so an intermediate artifact would be pure overhead.
5. Once Tom says to proceed, Claude Code commits and pushes `dataforge-standards` directly, following Section 5d's branch discipline and using the commit message format from Section 14e.
6. **Notion does not sync automatically** when this happens in Claude Code — see Section 14f. Do a short Claude.ai chat check-in afterward if the Notion row needs updating for this session, or update it by hand.

**Fallback path — Claude.ai chat only, no Claude Code available in the thread:** generate one CC prompt MD file per changed doc (Section 4's format/naming — `CC_[ShortDescription].md`), for Tom to paste into a separate Claude Code session, which writes to disk and commits. Use this only when there's no Claude Code session to write directly in.

**Why direct writing is the default now:** the original flow had Tom pre-drafting CC prompts (in a chat session, working from that session's own memory of file state) with specific AD/PD/ISSUE/BM numbers already baked in, while a separate Claude Code session was independently writing to the same files in parallel. That's a race condition, not a file-count problem — two different bugs found this way got assigned the same ISSUE number, and a batch of BestMethods entries were prompted to insert at a number three sessions' worth of entries had already passed. Direct writing collapses this to one point where "what's the next number" gets decided, immediately before it's used, so there's nothing to go stale.

No manual re-upload to Claude Project needed either way. GitHub commit = live for next session.

**Do not rely on screenshots to capture updated MD content — always get the text.**

### 5c. Rules for writing MD files
- **No duplicate information** — if a fact lives in TechnicalArchitecture, do not repeat it in SessionStarter. Reference it instead.
- **No sensitive data in MD files** — no API keys, tokens, passwords, connection strings. Use `[see local.settings.json]` or `[env var: VARIABLE_NAME]` as placeholders.
- **No screenshots for critical info** — tokens, API keys, emails, IDs must be copy/pasted as text, never captured in images.
- **Keep SessionStarter lean** — it is loaded every session and consumes tokens. Link to other docs for detail.
- **DecisionLog is append-only** — never edit or delete past entries; mark superseded decisions as `Superseded by [ID]`.

### 5d. Git Branch Discipline — all repos

**Applies to every repo Claude Code commits to — `dataforge-standards`
and every private project repo (AFAS, etc.) alike.** This section
exists because CC has, on more than one occasion, committed real
session work to a newly created feature/docs branch instead of `main`,
without being asked to — orphaning already-correct, already-tested
fixes off of `main` until caught manually via GitHub's UI.

**Before making any commit, CC must:**
1. Run `git branch --show-current` (or equivalent) and confirm the
   result is `main`.
2. If it is not `main`: run `git checkout main` and `git pull` first.
   Do not commit to whatever branch happens to be currently checked
   out without first confirming it's `main`.
3. Never create a new branch on Claude's or CC's own initiative. A
   branch is only created when Tom explicitly asks for one by name
   (e.g. for a deliberate PR-review workflow) — "let's make a branch
   for this" or similar, said in plain language, not inferred from
   context.
4. After committing, push directly to `origin main` — not to a
   feature branch, and not opening a PR unless Tom explicitly asked
   for a PR-based review flow for that specific piece of work.

**If CC finds itself already on a non-main branch it didn't expect**
(e.g. left over from a prior session, or created by mistake), it
should stop, tell Tom which branch it's on and why it looks
unexpected, and wait for Tom's direction rather than either committing
there or unilaterally switching and proceeding.

**Rationale:** This project has no CI/CD pipeline — a commit to a
branch other than `main` does not get deployed, does not show up in
next-session's `curl` fetch of the docs repo, and is invisible to any
future session unless someone happens to notice it in GitHub's branch
list. The cost of checking `git branch --show-current` before every
commit is trivial; the cost of a silently orphaned branch has already
required two rounds of manual cleanup this week alone.

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

**Prefixes by project (not by client — see Section 13a):**
| Project | Prefix |
|---------|--------|
| UP Golf | DEC |
| Club Golf | CGAD |
| AFAS | AFAS |
| HQ Dashboard | AD / DD / PD |

*When a new project starts under an existing client, it gets its own prefix — never reuses another project's prefix, even under the same client. E.g. if MicroSynergies resumes: `ask-the-docs` → ATD, `tech-team-wiki` → TTW, `knowledge-hub` → KH.*

**Always check the DecisionLog for the next available ID before adding an entry.**

**This is the default format for a new project or a project that already uses it.** Not every live project follows the per-decision `### PREFIX-NNN` header format — AFAS's DecisionLog logs decisions in session-grouped `Date | Decision | Rows` tables instead, with no per-decision ID. When a project has an established convention that differs from this section, match that project's own convention rather than forcing this template onto it retroactively (see the same hedge in Section 5b step 3).

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

**Handled automatically via the Notion connector (Claude.ai chat only — see Section 14f):**
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

## 14. "Update the Docs" — Full Procedure

**This is the single, complete procedure for the "Update the docs" trigger.**
Everything that happens as a result of that phrase lives in this section —
if you're looking for what to do when Tom says it, start and end here;
14d points out to Section 5b for file-writing mechanics and nowhere else.

### 14a. Trigger — read this before doing anything else

**Only when Tom explicitly says "Update the docs"** (or unambiguous
equivalent, e.g. "sync the docs"). Not before, not proactively, not because
a new decision/issue/lesson came up mid-session that "clearly belongs in a
doc eventually" — note that it happened, keep working, and fold it in once
the trigger actually fires.

**Never ask whether to update docs, or offer to.** Asking mid-session
("want me to log this now?") is its own form of the violation this section
prohibits, not a safe workaround of it — wait for the explicit trigger
silently, the same as any other deferred action.

**Only update MD files that actually changed this session** — not all 7
every time. The standard file set is Section 5a's list; here's the same
list with the update condition for each:
- `SessionStarter.md` — almost always needs updating (status, completed items, next priorities)
- `ProjectRoadmap.md` — update if phases or tasks changed
- `DecisionLog.md` — append only if new decisions were made (append-only — see 5c)
- `IssuesTracker.md` — update if issues were opened, deferred, or resolved
- `TechnicalArchitecture.md` — update only if stack or data models changed
- `BestMethods.md` — update only if new lessons were learned
- `TimeLog.md` — always update (duration + one-sentence summary)

### 14b. Which case applies — same fork as Section 4a, applied to docs

Just as important to get right here as it is for code, and for the same
underlying reason: **Claude.ai chat has no git write access to
`dataforge-standards`** (it can `curl`/read raw files, but cannot commit)
— so which case applies determines whether a handoff step is even possible
to skip.

**Case 1 — Claude Code did the session's actual work** (an AFAS coding
session, a debugging session, a session like this one). Claude Code
already knows everything that happened. When Tom says "update the docs,"
skip straight to 14d — no handoff file needed, nothing missing.

**Case 2 — Claude.ai chat did the session's actual work** (a design,
discussion, or decision-making conversation with no repo access involved).
Chat cannot write the files itself no matter how thoroughly it understood
the session — it has to hand off to a Claude Code session that can. Go to
14c first.

### 14c. Chat's handoff — facts for the docs, plus chat's own Session Summary (Case 2 only)

When "update the docs" is said in a Claude.ai chat session that did the
session's actual work, chat produces **one self-contained file**,
delivered the same way as a CC prompt (Section 4b's
`create_file`/`present_files` mechanic) but named distinctly —
`SessionSummary_[Project]_[YYYY-MM-DD].md` — so it doesn't get confused
with a single targeted code-change prompt. It has two parts, held to two
different standards:

**Part 1 — facts for the doc files, not pre-drafted doc text.** This
mirrors 5b step 1 exactly — state what happened (decisions made, issues
found/resolved, lessons learned, specific numbers/dollar amounts/file
names, whatever is factually true of the session) and do **not**
pre-draft exact replacement text for DecisionLog/IssuesTracker/
BestMethods/SessionStarter/TechnicalArchitecture, assume line numbers, or
pre-assign AD/PD/ISSUE/BM numbers. Claude Code will read each affected
file's live content itself in 14d and decide how to apply the facts — a
pre-drafted "here's what DecisionLog should now say" section skips that
live-verification step and risks writing something that no longer matches
the file Claude Code actually finds (see 14e for what happens when a fact
here turns out not to hold up against live data — it happens, and it's
not a failure of this process, it's the process catching something).
Cover everything relevant, even if it doesn't obviously map to one file —
anything left out effectively didn't happen, from Claude Code's side of
the handoff.

**Part 2 — chat's own draft Session Summary (the 14e format: a bulleted
list of what was accomplished).** Unlike Part 1, chat should draft this in
full — it's not a live file being verified against, it's a narrative
recap, and chat has the most complete view of the session to write it
from. **Use this draft to run 14f's Notion sync immediately, in this same
chat turn — don't wait for Claude Code.** Claude Code will independently
produce its own commit-message-level summary when it actually writes the
files in 14e (it has to; only it knows what ended up on disk after
verification), and that version can end up worded slightly differently
from what's already in Notion if verification surfaced a correction —
that's an acceptable, low-stakes gap. Notion is a casual log for people,
not a source of truth any future session reads back from; the commit
history and the doc files themselves are.

### 14d. Write the files

Follow Section 5b's mechanics exactly: **default is a Claude Code session
with a local `dataforge-standards` clone reading live content and writing
directly to disk** (no intermediate `CC_*.md` prompt file). If the session
originated in chat (Case 2), the "plain-language summary" 5b step 1
describes **is** the Session Summary file from 14c — Claude Code reads it,
then reads each affected file's live content itself exactly as 5b already
specifies. The handoff file is an input to that verification, not a set of
instructions to apply without checking.

No Claude Project re-upload needed either way — Claude fetches latest via
URL at next session start (or via an explicit "Read the docs," Section 3d).

Once Tom says to proceed, commit and push `dataforge-standards` directly,
following Section 5d's branch discipline (confirm `main` before committing)
and the commit message format in 14e below.

### 14e. Finalize the commit message (and the Session Summary, if there's no chat draft to start from)

**The commit message is always produced by whoever performs 14d's write —
normally Claude Code — regardless of whether chat drafted anything in
14c.** Only the session that actually wrote the files knows what ended up
on disk after live verification, so the commit message has to be generated
fresh at this point, not copied from a draft.

**The Session Summary is different: if chat already drafted one in 14c
(Part 2), start from that** rather than writing a new one from scratch —
it was already well-informed at the time, and re-deriving it independently
just duplicates work chat already did. Adjust it only if 14d's live-file
verification turned up something the draft got wrong or couldn't have
known (a stale count, a claim the live data doesn't support, a file that
changed since the handoff was written) — **when that happens, say so
plainly rather than silently editing it**, since it's evidence the process
caught something real. If there's no chat draft (Case 1, or a Case 2
handoff that only had Part 1), draft the Session Summary here from
scratch, the same as before.

Both generated automatically as part of this same turn — no separate
prompt needed, and not optional.

**Session Summary** — a bulleted or numbered list of what was accomplished,
not a single prose paragraph. One item per fix/finding/decision, written
plainly (what happened, not just an issue number). Optionally lead with one
short sentence of framing if the session had a clear throughline, but the
substance goes in the list. Covers: what was built or fixed, any
significant debugging/environment issues resolved, current status of the
project.

**GitHub commit message** — conventional commit format:
```
<type>: <short summary line>

<bullet: what changed>
<bullet: what changed>
<bullet: why or outcome>
```
Types: `feat` (new feature), `fix` (bug fix), `chore` (config/docs/tooling),
`refactor` (code change, no behavior change).

**TimeLog.md's summary line** should match the Session Summary in tone and
brevity — one sentence capturing the session's main accomplishment. This is
the same TimeLog update referenced in 14a's file list, not a separate step.

### 14f. Sync Notion (chat sessions only)

Notion is connected via the Notion MCP connector **in Claude.ai chat only.**
Claude Code has no access to it — connecting a service in claude.ai does
not extend to Claude Code, which would need its own separate MCP server
setup to reach Notion at all. **A Claude Code session doing 14d does not
have Notion access and skips this entirely.**

**Two different timings depending on which case applied in 14b:**

- **Case 2 (chat did the session):** already done. Chat ran this step back
  in 14c, immediately, using its own Part 2 draft — no round trip. Nothing
  further needed here once 14d/14e finish writing and committing.
- **Case 1 (Claude Code did the session):** chat wasn't part of the work
  and has nothing to draft from, so it needs 14e's finalized Session
  Summary handed to it. Tom brings that text to a Claude.ai chat session
  (the same one or a fresh one — Notion access is the only requirement)
  specifically to run this step, and chat uses it **verbatim** — it should
  not try to reconstruct its own summary of a session it wasn't part of.

When running in chat (either case above), fully automatic, no confirmation needed:
1. Update the `Last Updated` date property on the project's Notion page
   (Projects data source, page ID `37462bde-ae68-80a2-ab5a-c2f6fcbae3c6`
   for Kids HQ) to the session's date, via `notion-update-page` with
   `update_properties`, setting `date:Last Updated:start` to today and
   `date:Last Updated:is_datetime` to `0`. Happens every time docs are
   updated at session end via chat — not just when something notable
   changed — so the Notion row always reflects the most recent session date.
2. Update Status if the session moved the project to a new phase (e.g.
   Planning → In Progress, or In Progress → Complete).
3. Log the session to the shared "Daily Summaries" database (data source id
   `2284bd95-1027-4c7a-938a-e56a66dfa60d`) via `notion-create-pages`, with
   `Name` = `"{Project Name} — {session date, YYYY-MM-DD}"`, `Date` = the
   session's date, `Project` = relation to this project's row in the
   Projects database, and page body content = the Session Summary being
   used for this case (chat's own Part 2 draft for Case 2, or 14e's
   finalized text for Case 1) — don't write a third, separate summary.
   Applies to every project following this protocol, every time "Update the
   docs" runs, not just Kids HQ.

**Before the first automatic write in a new area of Notion**, confirm the
actual database/property names by reading the schema via the connector
rather than guessing — property names (e.g. "Local Folder" vs "Local Path")
must match exactly or the write will silently create a new property
instead of filling the existing one.

(Kickoff's Notion automation, Section 13, is a separate trigger from
"Update the docs" but follows the same chat-only constraint.)

### 14g. Never do (specific to this trigger)

- Never update, generate, or modify any MD files unless explicitly told "Update the docs" (14a)
- Never ask whether to update docs, or offer to, before that phrase (14a)
- Never touch all 7 files reflexively — only ones that actually changed (14a)
- Never pre-draft exact DecisionLog/IssuesTracker/BestMethods/etc. replacement text in the Part 1 facts handoff — only Part 2 (the Session Summary) is meant to be drafted in full (14c)
- Never generate a `CC_*.md` prompt file for a doc update when a Claude Code session can write directly (14d)
- Never silently edit a chat-drafted Session Summary during 14e without saying so — a correction is worth surfacing, not hiding (14e)
- Never make Case 1 skip the chat round-trip Notion needs, and never make Case 2 wait for one it doesn't need (14f)

---

## 15. What Claude Should Never Do — Quick-Reference Index

Every item here is stated in full, once, in its owning section. This is a
scan list, not a second copy — if you're unsure of the exact rule, follow
the link, don't rely on the one-line summary alone.

- Docs: see 14g in full — don't update/ask-to-update outside the explicit trigger; chat can't write the repo, so a Case 2 doc handoff (14c) always goes through Claude Code, but chat's own Session Summary draft goes to Notion immediately, no round trip
- Code: never make changes unless explicitly asked — no proactive fixes, no "while I'm here" edits
- Security: never write sensitive values into any file or chat response, never read them from screenshots — Section 6
- Session start: never start working before all session docs are fetched via `curl` (not `web_fetch`) — Section 3b
- Debugging: never stack multiple speculative changes at once — Section 10
- Firestore: never inline collection/doc refs outside `firestorePaths.js` — Section 8
- PowerShell: never use `&&` chaining — Section 7
- PWA installs: never `npm install` without `--legacy-peer-deps` — Section 7
- MD files: never duplicate information across them — reference, don't repeat — 5c
- DecisionLog: never re-litigate a decision logged with an Active status — Section 11
- Client relationships: never encode one into a repo name, docs folder, or DecisionLog prefix — Section 13a
