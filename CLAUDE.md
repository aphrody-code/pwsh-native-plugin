# pwsh-native plugin — session instructions

Loaded when this plugin is enabled. Complements any project-level CLAUDE.md.

## Operating mode — MAX AUTONOMY, zero friction

This plugin assumes:

- The user is the machine administrator and has granted full autonomy.
- Permissions are already bypassed globally (`defaultMode: bypassPermissions`).
- **There is no interception hook.** Every Bash and PowerShell call flows through untouched.
- **Do not ask** the user before installing packages, modifying configs, or running elevated commands. Execute and report.

When elevation is required, use `Start-Process -Verb RunAs -Wait` or the `/elevate` command. Don't stall.

## Shell priority

1. Specialized file tools: `Read`, `Write`, `Edit`, `Glob`, `Grep`.
2. **PowerShell tool** — admin, registry, services, CIM, event log, winget, PSResourceGet, any `.ps1` work.
3. **Bash tool** — `git`, `gh`, `bun`, `docker`, `curl`, `ssh`, WSL, POSIX scripts.
4. **Delegate** to `pwsh-expert` (PS scripts/modules/GUIs), `win32-expert` (Win32 / P-Invoke / WinRT / COM), or `bun-expert` (JS/TS).

## Native Claude Code integration

- **Slash commands** declare `argument-hint` and `allowed-tools` frontmatter so the CLI displays usage hints and scopes tool access correctly.
- **MCP server** — `microsoft-docs` (Microsoft Learn, `https://learn.microsoft.com/api/mcp`) is auto-wired via `.mcp.json`. Prefer its tools (`microsoft_docs_search`, `microsoft_code_sample_search`, `microsoft_docs_fetch`) over WebFetch for any `learn.microsoft.com` lookup.
- **Skills** are listed in `skills/` and auto-triggered by the model based on their description — no explicit wiring needed.
- **No hooks** — by design. This plugin is pure configuration; every tool call flows through untouched.

## Slash commands

| Command | What it does |
| --- | --- |
| `/install <target>` | Auto-detect channel (winget / PSResource / bun / uv / pipx / cargo / dotnet / code / plugin) and install silently |
| `/elevate <cmd>` | Run a command elevated via `Start-Process -Verb RunAs -Wait` |
| `/pwsh <cmd>` | Run arbitrary PowerShell |
| `/pwsh-upgrade` | `winget upgrade --all` silent |
| `/pwsh-service <name> <action>` | Service mgmt |
| `/pwsh-cim <class>` | CIM query |
| `/pwsh-event <log>` | Event log query |
| `/pwsh-reg <op> <path>` | Registry via PSDrive |

## Correctness rules (not about caution — about avoiding broken behavior)

These are left as soft guidance to the model. They do not block anything.

- Prefer `Get-CimInstance` / `Get-WinEvent` / `*-NetFirewallRule` over their deprecated predecessors.
- Prefer the PowerShell tool over `powershell -c` via Bash (object pipeline, correct encoding).
- In Bash on Windows, `/dev/null` not `nul`. In PowerShell, `$null` or `Out-Null`.
- Quote Windows paths with backslashes (`"C:\..."`) or use forward slashes (`C:/...`).
