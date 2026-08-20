# New Project Kickoff — DataForge

**How to use this:** Fill in the fields below, then paste the whole "Prompt to Claude" block into a brand-new Claude.ai chat (this happens *before* a Claude Project exists for the new work — a plain chat is fine).

---

# New Project Kickoff — DataForge

**How to use this:** Fill in the six bracketed fields at the top of the
"Prompt to Claude" block below, then paste the *entire* block into a
brand-new Claude.ai chat (this happens *before* a Claude Project exists for
the new work — a plain chat is fine). Everything below the fields refers
back to them — you only fill each one in once.

---

## Prompt to Claude

```
New DataForge project kickoff.

Project name:          [e.g. "Caddy Tracker"]
One-line description:  [what it does, who it's for]
Stack:                 [e.g. React + Vite + Firebase | Azure Functions + Python | Node.js + Express]
GitHub repo name:      [e.g. caddy-tracker — will be private]
Local folder:          C:\Dev_Projects\[repo name]
OneDrive ref folder:   C:\Users\tjunk\OneDrive\Documents\_DATAFORGE\_CLIENTS\[ClientName]\   (client work only — delete this line for personal projects)
DecisionLog prefix:    [e.g. CT]

Follow the DataForge New Project Checklist using the fields above
everywhere a value is needed below. Do the following now:

1. Generate all 7 standard MD files for this project, pre-filled with
   what you know so far from the fields above (leave the rest as clear
   placeholders): SessionStarter.md, TechnicalArchitecture.md,
   ProjectRoadmap.md, DecisionLog.md, IssuesTracker.md, BestMethods.md,
   TimeLog.md. These are destined for [GitHub repo name]/ inside the
   dataforge-standards repo, not the project's own private repo.

2. Generate the Project Instructions block I'll paste into this
   project's new Claude Project — the curl-fetch session-start/session-end
   protocol, with [GitHub repo name] substituted into every URL.

3. Present all of the above as downloadable files.

4. Do this automatically via the Notion connector, no confirmation needed
   (Section 17 of the Master Protocol):
   - Create the project row in the Notion Projects database
   - Set Status to Planning or In Progress
   - Add services to the Notion Services database, linked to this project
   - Add the Local folder and OneDrive ref folder paths to the project row
   - Add the GitHub repo URL to the project row
   If this is the first time writing to a given Notion database/property in
   this conversation, read the schema via the connector first rather than
   guessing property names.

5. Then walk me through the remaining checklist items one at a time,
   waiting for my confirmation before moving to the next — these can't be
   automated because they happen outside Notion:
   - Create the GitHub repo (private), with a /docs folder only if this
     project keeps any client-facing docs outside dataforge-standards
   - Commit the 7 MD files to dataforge-standards/[GitHub repo name]/
   - Create the Claude Project in claude.ai and paste in the
     Project Instructions block from step 2
   - Confirm .gitignore covers .env, local.settings.json, node_modules
     (NOT the whole .claude folder — .claude/skills should stay committed
     if this project uses any project-level Skills)
   - If this is a PWA project (React/Vite/Firebase): copy the
     pwa-firebase-rules skill into .claude/skills/ in the new repo
   - Confirm no secrets exist in any committed file

Stop and confirm with me after each item in step 5 rather than assuming
it's done. Step 4's Notion writes don't need confirmation, but tell me
what was written so I can spot-check it in Notion afterward.
```

---

## Notes for whoever reads this later

- This assumes the doc-storage decision from Master Protocol Section 3a (docs live in `dataforge-standards`, not the project's own repo). If that ever changes, update this template and the checklist together — they drifted apart once already.
- The `pwa-firebase-rules` Claude Code skill only makes sense for React/Vite/Firebase projects — don't copy it into AFAS or MicroSynergies-style repos.
- `cc-prompt-delivery`, `powershell-windows-rules`, and `session-docs-protocol` are personal skills (`~/.claude/skills/`) and don't need anything project-specific done — they'll already be active in Claude Code for this repo automatically.
