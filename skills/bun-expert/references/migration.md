# Migrating from npm / pnpm / yarn to bun

On this machine, `npm`, `pnpm`, `yarn`, `npx` are **forbidden** (denied at the permissions layer). Use this mapping.

## Command mapping (most common)

| From | To |
| --- | --- |
| `npm init` / `yarn init` / `pnpm init` | `bun init` |
| `npm install` / `yarn` / `pnpm install` | `bun install` |
| `npm ci` | `bun install --frozen-lockfile` |
| `npm install <pkg>` / `yarn add <pkg>` / `pnpm add <pkg>` | `bun add <pkg>` |
| `npm install -D <pkg>` / `yarn add -D <pkg>` / `pnpm add -D <pkg>` | `bun add -d <pkg>` |
| `npm install --save-peer <pkg>` | `bun add --peer <pkg>` |
| `npm install --save-optional <pkg>` | `bun add --optional <pkg>` |
| `npm install -g <pkg>` | `bun add -g <pkg>` (or `bun install -g <pkg>`) |
| `npm uninstall <pkg>` / `yarn remove <pkg>` / `pnpm remove <pkg>` | `bun remove <pkg>` |
| `npm update` / `yarn upgrade` / `pnpm update` | `bun update` |
| `npm update --latest` / `yarn upgrade --latest` | `bun update --latest` |
| `npm outdated` / `yarn outdated` / `pnpm outdated` | `bun outdated` |
| `npm audit` / `yarn audit` / `pnpm audit` | `bun audit` |
| `npm run <script>` / `yarn <script>` / `pnpm <script>` | `bun run <script>` or `bun <script>` |
| `npx <cli>` / `yarn dlx <cli>` / `pnpm dlx <cli>` | `bun x <cli>` |
| `npm publish` / `yarn publish` / `pnpm publish` | `bun publish` |
| `npm link` / `yarn link` / `pnpm link` | `bun link` |
| `npm pack` / `yarn pack` / `pnpm pack` | `bun pm pack` |
| `npm ls` / `yarn list` / `pnpm list` | `bun pm list` / `bun list` |
| `npm why <pkg>` / `yarn why <pkg>` / `pnpm why <pkg>` | `bun why <pkg>` |
| `npm view <pkg>` / `pnpm info <pkg>` | `bun info <pkg>` |
| `npm version <inc>` / `yarn version` / `pnpm version` | `bun pm version <inc>` |
| `npm exec <cli>` | `bun x <cli>` |

## Lockfile

- Bun writes `bun.lock` (text, committable).
- Delete `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml` after first `bun install` — bun reads them on first run to seed `bun.lock`, then no longer needs them.
- In CI: `bun install --frozen-lockfile` is the equivalent of `npm ci` / `yarn install --immutable` / `pnpm install --frozen-lockfile`.

## Scripts in `package.json`

Already compatible. Run with `bun run <name>`. Bun ignores `npm-run-all`, `concurrently`, etc. unless you install them — but it has `--parallel` / `--sequential` / `--workspaces` built in.

### Replace common scripts

```json
// Before
{ "scripts": {
    "dev":   "concurrently \"npm:build:*\" \"npm:serve\"",
    "build": "npm-run-all build:ts build:css",
    "test":  "jest"
}}

// After
{ "scripts": {
    "dev":       "bun run --parallel build:ts build:css serve",
    "build":     "bun run --sequential build:ts build:css",
    "build:ts":  "bun build ./src --outdir dist",
    "build:css": "tailwindcss -i ./src/app.css -o ./dist/app.css",
    "test":      "bun test"
}}
```

## Special dependency protocols

| Protocol | Works in bun? | Notes |
| --- | --- | --- |
| `file:./local-pkg` | ✅ | Copy-install. |
| `link:./local-pkg` | ✅ | Symlink-install (editable). |
| `workspace:*` / `workspace:^` / `workspace:~` | ✅ | Monorepo siblings. |
| `npm:<realname>@<range>` | ✅ | Rename-install. |
| `github:user/repo` / `git+https://…` | ✅ | Git tarballs. |
| `catalog:` / `catalog:<name>` | ✅ | Shared version pins (see package-manager.md). |

## Tools that may assume npm/yarn

Some tools hardcode "npm" or "yarn" commands:

- **`create-<template>` packages** (e.g. `create-next-app`, `create-vite`): they usually accept `--use=bun` now. Or run via `bun create <template>` directly.
- **`husky`**: works with bun; run `bun x husky init`.
- **`lint-staged`**: works; just use `bun x lint-staged` in the hook.
- **CI examples** (Vercel, Netlify, Render): most have first-class bun support now. If not, set the install command to `curl -fsSL https://bun.com/install | bash && ~/.bun/bin/bun install` and the build to `~/.bun/bin/bun run build`.
- **VS Code debugger**: works out of the box; launch config uses `bun` as runtime.

## Gotchas

- **Post-install scripts don't run** by default (security). Add packages to `trustedDependencies` in `package.json` to allow them.
- **Peer-dep resolution is stricter** than npm's historically-loose style. Some older React-ecosystem packages need explicit `--legacy-peer-deps` behavior — bun has `--peer` for adding, but doesn't ignore unmet peers silently.
- **`bun run` prints the command line** by default — pass `--silent` to suppress.
- **`bun x <cli>` caches into `~/.bun/install/cache`** — subsequent runs are instant. For fully fresh runs in CI, `bun x --no-cache <cli>`.
- **Some native deps** (e.g., `bufferutil`, `sharp`, `canvas`) may need `trustedDependencies` or prebuilt binaries. Check the dep's docs.
- **Compared to pnpm**, bun does NOT enforce strict isolation by default; hoists aggressively. If you want pnpm-like strictness: `bun install --node-linker=isolated` (check current availability).
- **Switching from npm-workspaces to bun**: `workspaces` field format is identical. No changes needed.
