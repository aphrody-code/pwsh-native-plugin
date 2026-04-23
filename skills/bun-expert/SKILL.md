---
name: bun-expert
description: Bun (JavaScript/TypeScript runtime, package manager, bundler, test runner) expertise. Use when writing, debugging, packaging, or running JS/TS with bun — install/add/remove deps, run scripts, test, build bundles, compile standalone executables, scaffold projects with bun create / bun init, publish packages. Handles migration from npm/pnpm/yarn (which are forbidden on this machine). Verifies against live bun docs when accuracy is critical.
---

# Bun expert

Bun is a fast all-in-one toolkit for JavaScript and TypeScript: runtime, package manager, bundler, test runner, and shell executor. On this machine, **bun replaces `node`, `npm`, `pnpm`, `yarn`, and `npx` entirely** — all five are denied at the permissions layer. This skill covers the full bun CLI surface and common workflows.

Installed: bun 1.3.13-canary on Windows 11 (x64).

---

## Quick reference — top-level commands

| Area | Command | Alias |
| --- | --- | --- |
| **Run a file / script** | `bun run ./file.ts` or `bun run <script>` (from `package.json`) | — |
| **Package CLI execution** | `bun x <cli>` (auto-install + run) | `bunx` |
| **REPL** | `bun repl` | — |
| **Run unit tests** | `bun test` | — |
| **Execute shell script** | `bun exec <file>` (uses Bun's portable shell) | — |
| **Install deps** | `bun install` | `bun i` |
| **Add a dep** | `bun add <pkg>` | `bun a` |
| **Add dev dep** | `bun add -d <pkg>` | — |
| **Remove a dep** | `bun remove <pkg>` | `bun rm`, `bun r` |
| **Update deps** | `bun update [<pkg>]` | — |
| **Audit vulnerabilities** | `bun audit` (or `bun pm scan`) | — |
| **List outdated** | `bun outdated` | — |
| **Package metadata** | `bun info <pkg>` | — |
| **Why is X installed?** | `bun why <pkg>` | — |
| **Link local package** | `bun link` / `bun unlink` | — |
| **Patch a dep** | `bun patch <pkg>` | — |
| **Pkg utilities** | `bun pm <pack\|bin\|scan\|whoami\|version\|pkg\|why\|list>` | — |
| **Bundle** | `bun build <entry> --outdir dist` | — |
| **Standalone exe** | `bun build <entry> --compile --outfile app.exe` | — |
| **Scaffold project** | `bun init` (empty) or `bun create <template>` | `bun c` |
| **Publish to npm** | `bun publish` | — |
| **Upgrade bun itself** | `bun upgrade` (stable) or `bun upgrade --canary` | — |

---

## The forbidden-node/npm rule

On this machine **`node`, `npm`, `pnpm`, `yarn`, `npx` are all denied** via `permissions.deny` in `~/.claude/settings.json`. Always translate:

| Forbidden | Use |
| --- | --- |
| `node script.js` | `bun script.js` (bun runs JS and TS natively) |
| `#!/usr/bin/env node` shebang | `#!/usr/bin/env bun` |

Package manager translations:

| If the user/docs say | Use instead |
| --- | --- |
| `npm install` | `bun install` |
| `npm install <pkg>` | `bun add <pkg>` |
| `npm install -D <pkg>` / `npm install --save-dev <pkg>` | `bun add -d <pkg>` |
| `npm uninstall <pkg>` | `bun remove <pkg>` |
| `npm update` | `bun update` |
| `npm run <script>` | `bun run <script>` (or just `bun <script>` if no name conflict) |
| `npx <cli>` / `pnpm dlx <cli>` | `bun x <cli>` |
| `yarn`, `pnpm install` | `bun install` |
| `yarn add`, `pnpm add` | `bun add` |
| `npm ci` | `bun install --frozen-lockfile` |
| `npm audit` | `bun audit` |
| `npm publish` | `bun publish` |
| `package-lock.json` / `yarn.lock` / `pnpm-lock.yaml` | `bun.lock` (text, committed) |

Bun reads `package.json` natively and is compatible with npm / yarn lockfiles on first install (it migrates them to `bun.lock`).

---

## Running TypeScript & JSX — no build step

```bash
bun ./src/index.ts         # runs TS directly, no tsc needed
bun ./src/App.tsx          # JSX/TSX work out of the box
bun --hot ./server.ts      # hot-reload on file change (server-friendly)
bun --watch ./worker.ts    # full restart on file change
bun -e 'console.log(Bun.version)'   # inline eval
bun -p '2 + 2'             # inline eval + print result
```

Runtime knobs (the ones you'll actually use):
- `--hot` — keep process alive across reloads (preserves open sockets). Prefer for servers.
- `--watch` — full restart. Prefer for short-lived scripts.
- `--smol` — use less memory (GC more often). For low-RAM environments.
- `--preload=<file>` (alias `-r`, `--require`, `--import`) — import before user code.
- `--env-file=<path>` — load env from file(s). Default auto-loads `.env`, `.env.local`, etc.
- `--bun` — force a bin/script to use bun (not node) via node shim symlink.
- `--inspect=<port>` — enable the debugger (V8 Inspector protocol).
- `--install=auto|fallback|force` — auto-install packages on run. `-i` = shortcut for `fallback`.
- `--cpu-prof[-md]` / `--heap-prof[-md]` — built-in profilers. The `-md` variants emit markdown for LLM analysis.

---

## Key workflows

### Workflow: bootstrap a new project

```bash
bun init -y                            # empty TS project (package.json + tsconfig.json + bunfig.toml)
bun init --react=tailwind my-app       # React + TailwindCSS starter
bun create next-app my-app             # from npm create-<name> template
bun create vercel/next.js/examples/api-routes  my-app   # from GitHub path
bun create ./MyComponent.tsx           # turn a React component into a runnable app
```

### Workflow: standalone executable

```bash
# Bundle + compile a single-file exe (includes the runtime)
bun build ./src/cli.ts --compile --outfile mycli.exe
# Minified release
bun build ./src/cli.ts --compile --production --outfile mycli.exe
# Cross-compile targets: browser | bun | node
bun build ./src/lib.ts --target bun --outdir dist
```

### Workflow: run tests

```bash
bun test                                      # all matching test files
bun test src/auth                             # path pattern
bun test -t 'authorize.*admin'                # regex on test name
bun test --coverage --coverage-reporter=lcov  # LCOV for CI
bun test --update-snapshots                   # update .snap files
bun test --bail=3 --randomize --seed=42       # stop at 3 fails, randomize w/ seed
```

### Workflow: monorepo / workspaces

Declare workspaces in the root `package.json`:
```json
{ "workspaces": ["packages/*", "apps/*"] }
```

Common commands:
```bash
bun install                                   # installs everything, hoists
bun add react -w apps/web                     # scope to one workspace
bun run --workspaces build                    # run "build" in every workspace
bun run -F 'apps/*' --parallel dev            # filter + parallel Foreman
bun outdated --workspaces
```

### Workflow: migration from npm/yarn/pnpm

```bash
bun install                 # reads existing lockfile, creates bun.lock
rm package-lock.json yarn.lock pnpm-lock.yaml
git add bun.lock
# Replace calls in CI / scripts: sed -i 's/npm run/bun run/g' …
```

---

## When to delegate — or consult references

Keep SKILL.md lightweight. For deeper dives, load these on-demand:

- **[references/package-manager.md](references/package-manager.md)** — full `install`/`add`/`remove`/`update`/`audit` flags, lockfile semantics, workspaces, catalogs, overrides, `trustedDependencies`, `patch`.
- **[references/runtime.md](references/runtime.md)** — Bun.serve, Bun.sql, Bun.redis, fetch, HTTP/2, env handling, import maps, FFI (`bun:ffi`), IPC, SQLite (`bun:sqlite`).
- **[references/bundler.md](references/bundler.md)** — all `bun build` flags, plugins API, externalization, tree-shaking, code-splitting, `--bytecode` cache, standalone-exec autoload flags.
- **[references/testing.md](references/testing.md)** — full `bun test` API: matchers, mocks, `spyOn`, timers, DOM via Happy DOM, snapshots, coverage, parallel/concurrent control.
- **[references/shell.md](references/shell.md)** — `Bun.$` tagged template (official API), `bun exec`, piping, redirection, builtins, interpolation safety.
- **[references/migration.md](references/migration.md)** — detailed npm/pnpm/yarn -> bun mapping, tricky cases (postinstall scripts, peer deps, overrides, protocols like `file:`, `link:`, `workspace:`).
- **[references/node-compat.md](references/node-compat.md)** — Node.js module compatibility matrix (full / partial / missing) and when to prefer `Bun.*` over `node:*`.
- **[references/docs-index.md](references/docs-index.md)** — complete URL map of `bun.com/docs/*` (runtime, pm, bundler, test, guides, networking, project). Use before WebFetching a generic URL.

---

## Verify when accuracy matters

Before quoting flags or behavior you're not 100% sure of:

1. `bun --help` / `bun <subcommand> --help` — source of truth, current version.
2. `WebFetch https://bun.com/docs/<topic>` — official docs.
3. Check the bun source in a pinch: https://github.com/oven-sh/bun

If a feature is canary-only, say so explicitly. Current machine has `1.3.13-canary.1+f8d4254d9`.

---

## Common pitfalls

- **`bun add` doesn't auto-save dev deps** — use `-d` / `--dev` explicitly for tooling.
- **`--frozen-lockfile`** = CI-safe; will fail if `package.json` and `bun.lock` diverge. Use in CI, not local dev.
- **`bun test` finds files matching `*.test.*`, `*.spec.*`, `*_test.*`, `*_spec.*`** — not the same glob as Jest's default. Configure via `bunfig.toml` `[test] testNamePattern` if needed.
- **`bun x <cli>`** auto-installs into a global cache at first use. Subsequent calls are fast. For fully reproducible runs in CI, prefer `bun add -d <cli>` + `bun <cli>`.
- **`--compile` binaries embed the runtime** — expect 50-100 MB per exe. Good for CLIs, heavy for tiny tools.
- **Hot reload + stateful modules**: `--hot` preserves HTTP server sockets; top-level state persists. Re-init inside handlers, not at module scope.
- **Windows paths in `package.json` scripts** — use forward slashes or escape backslashes; bun's shell is POSIX-compatible.
- **`bun` as node drop-in** — most works, but some Node-specific internals (`vm` quirks, certain `fs.promises` edge cases, some `net` behaviors) may differ. Run failing tests with `node` to confirm the divergence before filing a bug.
