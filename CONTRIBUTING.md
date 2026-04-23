# Contributing to pwsh-native

Thanks for your interest. This plugin is deliberately small — it's a curated set of agents, skills, and slash commands for Windows-native Claude Code work, not a framework.

## Before opening a PR

1. **Open an issue first** for anything non-trivial (new agent, new command, behavior change). A quick alignment conversation saves rework.
2. **Scope**: one agent, command, or skill per PR. Bundling unrelated changes makes review harder.
3. **Validate the manifest**:
   ```powershell
   claude plugin validate ./pwsh-native
   ```

## Testing locally

```text
/plugin marketplace add C:/path/to/your/clone/pwsh-native-plugin
/plugin install pwsh-native@pwsh-native-marketplace
/reload-plugins
```

Exercise the commands you changed:

```text
/pwsh-native:pwsh $PSVersionTable
/pwsh-native:pwsh-cim Win32_OperatingSystem
```

## Style

- **Agents & commands**: keep frontmatter minimal and focused. One paragraph of description, then the prompt.
- **PowerShell in examples**: prefer modern cmdlets (`Get-CimInstance`, `Get-WinEvent`, `*-NetFirewallRule`, `Install-PSResource`). Never `Get-WmiObject`, `Get-EventLog`, `netsh`, `Install-Module`, `reg.exe`, `wmic`.
- **Markdown**: one `#` h1 per file, blank line between sections, inline `code` for commands, fenced blocks for multi-line examples.

## What we'll typically accept

- New slash commands for common admin tasks that fit the existing pattern.
- Corrections to `references/bash-to-pwsh.md` or `windows-admin-cmdlets.md`.
- Bug fixes in the manifest or command bodies.
- Documentation improvements.

## What we'll typically decline

- Adding a PreToolUse hook or any interception logic — this plugin is deliberately "zero friction, nothing blocked."
- Cross-platform abstraction. This is a **Windows-native** plugin; macOS/Linux users should use something else.
- Replacing the max-autonomy posture with a cautious one. If you need prompts, don't install this plugin.

## Reporting security issues

Please open a [GitHub issue](https://github.com/aphrody-code/pwsh-native-plugin/issues) for anything concerning. Given the plugin's max-autonomy posture, "runs destructive things silently" is the intended design, not a bug.
