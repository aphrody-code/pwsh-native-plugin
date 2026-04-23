# pwsh-native

> **Windows-native PowerShell-first plugin for [Claude Code](https://docs.claude.com/en/docs/claude-code)** — runs in max-autonomy mode. Claude installs, elevates, and executes without asking. No interception hook. No confirmation prompts.

[![Version](https://img.shields.io/badge/version-0.5.0-blue)](./.claude-plugin/plugin.json)
[![License: MIT](https://img.shields.io/badge/license-MIT-green)](./LICENSE)
[![Platform](https://img.shields.io/badge/platform-Windows%2010%20%7C%2011-0078d4)](https://www.microsoft.com/windows)
[![PowerShell](https://img.shields.io/badge/PowerShell-5.1%20%7C%207%2B-012456?logo=powershell&logoColor=white)](https://learn.microsoft.com/en-us/powershell/)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-v2.1.84%2B-d97757)](https://docs.claude.com/en/docs/claude-code)

---

## Install

```text
/plugin marketplace add aphrody-code/pwsh-native-plugin
/plugin install pwsh-native@pwsh-native-marketplace
/reload-plugins
```

That's it. No build step, no dependencies to fetch — the plugin is pure markdown + JSON manifests.

## Try it (30-second tour)

```text
# Query the OS via modern CIM (never WMI)
/pwsh-native:pwsh-cim Win32_OperatingSystem Caption,Version,OSArchitecture

# Service status
/pwsh-native:pwsh-service Spooler status

# Read the registry
/pwsh-native:pwsh-reg get HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion ProductName

# Install anything — auto-detects channel
/pwsh-native:install ripgrep

# Run something elevated (UAC click once, then flows)
/pwsh-native:elevate "Get-Service WinRM | Restart-Service"
```

---

## Why

- **Zero friction.** No PreToolUse hook, no permission prompts, no clarifying questions for obvious tasks. The assumption is you have admin on the box and you've granted full autonomy.
- **PowerShell-first tool choice.** Uses Claude Code's PowerShell tool (object pipeline, correct UTF-8 encoding) instead of `powershell -c` via bash.
- **Modern cmdlets only.** `Get-CimInstance` over `Get-WmiObject`. `Get-WinEvent` over `Get-EventLog`. `*-NetFirewallRule` over `netsh`. `Install-PSResource` over legacy `Install-Module`.
- **Universal installer.** `/install` auto-detects the right channel — winget, PSResourceGet, bun, cargo, pipx, `code --install-extension`, `claude plugin install`.
- **Silent elevation.** `/elevate` wraps any command in `Start-Process -Verb RunAs -Wait`. Only the UAC dialog itself requires a click — there is no programmatic bypass.
- **Specialist delegation.** Main-thread `pwsh-admin` delegates PS script authoring to `pwsh-expert`, Win32/P-Invoke/WinRT to `win32-expert`, and JS/TS/bun work to `bun-expert`.
- **Native Claude Code integration.** Ships an `.mcp.json` that auto-wires the Microsoft Learn MCP server (`https://learn.microsoft.com/api/mcp`) on install. Every slash command declares `argument-hint` and `allowed-tools` frontmatter for native CLI UX and proper tool scoping.

## What's in the box

| Component | Count | Items |
| --- | --- | --- |
| **Main-thread agent** | 1 | `pwsh-admin` — installs freely, elevates via UAC, never asks. Select via your user `~/.claude/settings.json`. |
| **Specialist subagents** | 4 | `pwsh-expert`, `win32-expert`, `bun-expert`, `pwsh-main` (alt general-purpose). |
| **Skills** | 4 | `powershell-expert`, `microsoft-docs`, `microsoft-code-reference`, `bun-expert`. |
| **Slash commands** | 8 | `/install`, `/elevate`, `/pwsh`, `/pwsh-upgrade`, `/pwsh-service`, `/pwsh-cim`, `/pwsh-event`, `/pwsh-reg`. |
| **MCP servers** | 1 | `microsoft-docs` (Microsoft Learn HTTP MCP) — auto-wired via `.mcp.json`. |
| **References** | 2 | `bash-to-pwsh.md` (idiom map), `windows-admin-cmdlets.md`. |

## Slash command reference

| Command | Purpose | Example |
| --- | --- | --- |
| `/install <target>` | Universal installer (auto-channel detection) | `/install ripgrep` |
| `/elevate <cmd>` | Run a command elevated via UAC | `/elevate "sc.exe config WinRM start= auto"` |
| `/pwsh <cmd>` | Run arbitrary PowerShell | `/pwsh Get-Process -Name code` |
| `/pwsh-upgrade` | `winget upgrade --all` silent | `/pwsh-upgrade` |
| `/pwsh-service <name> <action>` | Service management | `/pwsh-service Spooler restart` |
| `/pwsh-cim <class> [fields]` | CIM query (preferred over WMI) | `/pwsh-cim Win32_LogicalDisk` |
| `/pwsh-event <log> [count]` | Event log query (`Get-WinEvent`) | `/pwsh-event System 20` |
| `/pwsh-reg <op> <path> [name]` | Registry via PSDrive | `/pwsh-reg get HKCU:\Environment Path` |

### `/install` — channels auto-detected

| Target type | Channel |
| --- | --- |
| Claude plugin | `claude plugin install` |
| PowerShell module | `Install-PSResource -Scope CurrentUser -TrustRepository -Quiet` |
| npm package | **forbidden** — delegates to `bun-expert` (use `bun add`) |
| Python CLI | `pipx install` |
| Python library | `pip install` |
| Rust crate | `cargo install` |
| VS Code extension | `code --install-extension` |
| Windows app / system CLI | `winget install --silent --accept-source-agreements --accept-package-agreements --disable-interactivity` |

## Requirements

- **Windows 10 / 11**
- **[Claude Code](https://docs.claude.com/en/docs/claude-code) v2.1.84+** with the PowerShell tool enabled

Recommended `~/.claude/settings.json` for full autonomy:

```json
{
  "env": { "CLAUDE_CODE_USE_POWERSHELL_TOOL": "1" },
  "defaultShell": "powershell",
  "permissions": { "defaultMode": "bypassPermissions" },
  "skipDangerousModePermissionPrompt": true,
  "skipAutoPermissionPrompt": true
}
```

## Configuration

Set once during install or anytime via `/plugin config`:

| Key | Type | Purpose |
| --- | --- | --- |
| `ms_repos_path` | `directory` | Absolute path to local clones of Microsoft reference repos (`Windows-classic-samples`, `windows-rs`, `CsWin32`, `PowerShell-Docs`, etc.). Leave blank if none. Example: `C:/Users/you/ms-repos`. |

## Autonomy posture

| Scenario | Behavior |
| --- | --- |
| Claude needs a package | Installed silently via the right channel. |
| Command needs admin | Elevated via UAC (`Start-Process -Verb RunAs -Wait`), no pause. |
| User asks for something | Done. Reported. No clarifying questions unless truly ambiguous. |
| Destructive command | Executed. Nothing inspects or warns. |
| Footgun pattern (`curl \| sh`, `wmic`, `powershell -c` via bash) | Executed. No security layer. |

Correctness preferences are left as soft guidance to the model via `CLAUDE.md` — never enforced.

## File layout

<details>
<summary>Click to expand</summary>

```
pwsh-native/
├── .claude-plugin/plugin.json       # v0.5.0, userConfig: ms_repos_path
├── .mcp.json                        # auto-wires microsoft-docs MCP
├── CLAUDE.md                        # per-session posture (max autonomy)
├── agents/
│   ├── pwsh-admin.md                # main-thread, max autonomy
│   ├── pwsh-main.md                 # alt general-purpose Windows main
│   ├── pwsh-expert.md
│   ├── win32-expert.md
│   └── bun-expert.md
├── skills/
│   ├── powershell-expert/
│   ├── microsoft-docs/
│   ├── microsoft-code-reference/
│   └── bun-expert/
├── commands/                        # all commands declare argument-hint + allowed-tools
│   ├── install.md
│   ├── elevate.md
│   ├── pwsh.md
│   ├── pwsh-upgrade.md
│   ├── pwsh-service.md
│   ├── pwsh-cim.md
│   ├── pwsh-event.md
│   └── pwsh-reg.md
├── references/
│   ├── bash-to-pwsh.md              # 13-section idiom map
│   └── windows-admin-cmdlets.md
├── README.md
└── LICENSE
```

</details>

## FAQ / Troubleshooting

**Install fails with `userConfig.ms_repos_path.type: Invalid option`.**
Your local marketplace cache is stale. Run `/plugin marketplace update pwsh-native-marketplace`, then retry the install.

**`/reload-plugins` shows `0 plugins · 0 skills`.**
The plugin isn't installed for the current project. Run `/plugin install pwsh-native@pwsh-native-marketplace` first, then reload.

**Claude still asks me to confirm before running commands.**
Your `~/.claude/settings.json` doesn't have `permissions.defaultMode: "bypassPermissions"`. See [Requirements](#requirements).

**`/elevate` triggers UAC every time — can I bypass it?**
No. UAC consent is enforced by Windows itself; there is no programmatic bypass. The plugin minimizes friction (non-interactive temp script, `-Verb RunAs -Wait`) but the click is unavoidable.

**Why does the plugin refuse `npm install`?**
`npm`, `pnpm`, and `yarn` are globally forbidden on this machine — the `bun-expert` agent handles all JS/TS package management via `bun add`.

**Can I use this without max-autonomy mode?**
Yes, but you'll get permission prompts for every PowerShell and Bash call. The value prop is the zero-friction flow — running with permission prompts defeats the purpose.

## Validate the plugin manifest

```powershell
claude plugin validate ./pwsh-native
```

## Related projects

- [Claude Code](https://docs.claude.com/en/docs/claude-code) — the CLI this plugin extends
- [PowerShell](https://learn.microsoft.com/en-us/powershell/) — modern shell & automation framework
- [PSResourceGet](https://learn.microsoft.com/en-us/powershell/gallery/powershellget/overview) — modern PowerShell module manager
- [winget](https://learn.microsoft.com/en-us/windows/package-manager/) — Windows Package Manager
- [bun](https://bun.sh) — fast JS/TS runtime used in place of npm/pnpm/yarn

## Contributing

Issues and pull requests welcome. Please:

1. Open an [issue](https://github.com/aphrody-code/pwsh-native-plugin/issues) first to discuss non-trivial changes.
2. Keep PRs scoped — one agent, command, or skill per PR.
3. Run `claude plugin validate ./pwsh-native` before submitting.

## Changelog

See [CHANGELOG.md](../CHANGELOG.md).

## License

[MIT](./LICENSE) © [aphrody-code](https://github.com/aphrody-code)
