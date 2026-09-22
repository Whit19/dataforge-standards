SESSION START — required before any other action:

Fetch all session docs using the bash tool with `curl -s`.
Do NOT use web_fetch for raw.githubusercontent.com — it can silently
return a stale cached copy after a fresh commit, with no error.

| File | Purpose | Fetch |
|---|---|---|
| `MASTER_CLAUDE_PROTOCOL.md` | Always fetched first — overrides everything | `curl -s "https://raw.githubusercontent.com/Whit19/dataforge-standards/main/MASTER_CLAUDE_PROTOCOL.md"` |
| `SessionStarter.md` | Current status, next priorities, key decisions | `curl -s "https://raw.githubusercontent.com/Whit19/dataforge-standards/main/TNPL/SessionStarter.md"` |
| `TechnicalArchitecture.md` | Data models, stack, file structure | `curl -s "https://raw.githubusercontent.com/Whit19/dataforge-standards/main/TNPL/TechnicalArchitecture.md"` |
| `ProjectRoadmap.md` | Phased task list with status | `curl -s "https://raw.githubusercontent.com/Whit19/dataforge-standards/main/TNPL/ProjectRoadmap.md"` |
| `DecisionLog.md` | All decisions with rationale — prevents relitigating | `curl -s "https://raw.githubusercontent.com/Whit19/dataforge-standards/main/TNPL/DecisionLog.md"` |
| `IssuesTracker.md` | Open, deferred, resolved issues | `curl -s "https://raw.githubusercontent.com/Whit19/dataforge-standards/main/TNPL/IssuesTracker.md"` |
| `BestMethods.md` | Hard-won lessons — read before writing any code | `curl -s "https://raw.githubusercontent.com/Whit19/dataforge-standards/main/TNPL/BestMethods.md"` |
| `TimeLog.md` | Session time tracking | `curl -s "https://raw.githubusercontent.com/Whit19/dataforge-standards/main/TNPL/TimeLog.md"` |

Do not proceed until all fetches succeed. If any fail, stop and tell Tom.
Confirm what loaded: "TNPL docs loaded via curl. [status]. Ready."

MASTER_CLAUDE_PROTOCOL.md is authoritative once loaded and wins any disagreement with this snippet.
