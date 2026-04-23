# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.5.0] — 2026-04-23

### Added
- `.mcp.json` auto-wires the **Microsoft Learn MCP server** (`https://learn.microsoft.com/api/mcp`) when the plugin is enabled. Agents now reach the canonical docs via MCP tools (`microsoft_docs_search`, `microsoft_code_sample_search`, `microsoft_docs_fetch`) instead of WebFetch.
- Native Claude Code command frontmatter on all 8 slash commands: `argument-hint` (CLI usage display) and `allowed-tools` (per-command tool scoping).

### Changed
- Bumped plugin version to `0.5.0`.
- Agent frontmatter normalized — removed non-standard keys (`initialPrompt`, `memory`, `skills`) that Claude Code does not honor natively. Their content was moved into the agent body where the model actually reads it.
- `pwsh-admin` session kickoff (git status / recent commits / TODO files) migrated from the ignored `initialPrompt` frontmatter field into the agent body.
- `pwsh-admin` now explicitly delegates JS/TS to `bun-expert` in its delegation rules.
- Memory sections on each agent now reference the auto-memory system path (`~/.claude/projects/<project>/memory/`) instead of the deprecated `~/.claude/agent-memory/` tree.

### Removed
- Plugin-root `settings.json` (`{"agent": "pwsh-admin"}`) — not a valid Claude Code plugin file, silently ignored. Main-thread agent selection stays in the user's own `~/.claude/settings.json`.

## [0.4.0] — 2026-04-19

### Added
- Initial public release on GitHub.
- `pwsh-admin` main-thread agent with max-autonomy posture (silent installs, UAC elevation via `Start-Process -Verb RunAs`, no confirmation prompts).
- Specialist subagents: `pwsh-expert`, `win32-expert`, `bun-expert`, `pwsh-main`.
- Skills: `powershell-expert`, `microsoft-docs`, `microsoft-code-reference`, `bun-expert`.
- Slash commands: `/install`, `/elevate`, `/pwsh`, `/pwsh-upgrade`, `/pwsh-service`, `/pwsh-cim`, `/pwsh-event`, `/pwsh-reg`.
- References: `bash-to-pwsh.md` (13-section idiom map), `windows-admin-cmdlets.md`.
- User config: `ms_repos_path` (offline Microsoft docs clones).

### Fixed
- `userConfig.ms_repos_path` manifest validation — added required `type: "directory"` and `title` fields.
