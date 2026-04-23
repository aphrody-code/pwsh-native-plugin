# Bun's Node.js compatibility — quick reference

Bun implements most of Node's built-in modules and runs most npm packages unchanged. Status below is from the official compatibility matrix (<https://bun.com/docs/runtime/nodejs-compat>).

## Core modules

| Module | Bun status | Gotchas |
| --- | --- | --- |
| `node:os` | **Full** | 100% of Node's test suite passes. Use freely. |
| `node:path` | **Full** | 100% of Node's test suite passes. |
| `node:fs` / `node:fs/promises` | **Full** | 92% of Node's test suite. Some edge cases differ. |
| `node:net` | **Full** | — |
| `node:stream` | **Full** | — |
| `node:buffer` | **Full** | — |
| `node:events` | **Full** | 100% of Node's test suite passes. |
| `node:http` | **Full** | Client request body is buffered, not streamed. |
| `node:child_process` | **Partial** | Missing some process properties; IPC limitations. Prefer `Bun.spawn` where you can. |
| `node:crypto` | **Partial** | Missing `secureHeapUsed`, `setEngine`, `setFips`. |
| `node:https` | **Partial** | APIs work; `Agent` not always honored. |
| `node:cluster` | **Partial** | Handle passing limited to Linux; not battle-tested. |
| `node:worker_threads` | **Partial** | Missing stdin/stdout/stderr options. |
| `node:vm` | **Partial** | Core features work; missing `measureMemory`. |
| `node:tls` | **Partial** | Missing `tls.createSecurePair`. |
| `node:process` | **Partial** | Several stubs for newer APIs. |
| `node:util` | **Partial** | Missing call-site and transfer helpers. |

## When to prefer `Bun.*` over `node:*`

| Task | Node equivalent | Bun-native alternative (faster / simpler) |
| --- | --- | --- |
| Read file to string | `fs.promises.readFile(p, 'utf-8')` | `Bun.file(p).text()` |
| Write file | `fs.promises.writeFile` | `Bun.write(p, data)` |
| HTTP server | `http.createServer` | `Bun.serve` |
| Spawn child | `child_process.spawn` | `Bun.spawn` |
| Hash | `crypto.createHash` | `Bun.hash` / `new Bun.CryptoHasher` |
| Password hashing | (none built-in) | `Bun.password.hash` / `verify` |
| Env vars | `process.env` | `Bun.env` (typed via tsconfig) |
| SQLite | (requires `better-sqlite3`) | `bun:sqlite` (native, zero deps) |
| Postgres | (requires `pg`) | `Bun.sql` (native) |
| Redis | (requires `ioredis` etc.) | `Bun.redis` (native) |
| Shell | `child_process` + string munging | `Bun.$` tagged template |
| UUID | `crypto.randomUUID` | `crypto.randomUUID` (same) or `Bun.randomUUIDv7()` |
| Glob | (requires `fast-glob` / `glob`) | `Bun.Glob` |

## npm package support

Most packages "just work" — including React, Vue, Svelte, Express, Hono, Prisma, Drizzle, Zod, TanStack, most testing libs (with `bun test` as an alternative), most build tools (with `bun build` as an alternative).

Known friction points:
- **Native modules** that bundle prebuilt `.node` binaries per Node version (e.g., `sharp`, `better-sqlite3`, `canvas`) — usually work, sometimes need a rebuild. If not: `bun add -d <pkg>` then check `trustedDependencies`.
- **Packages that probe `process.versions.node` specifically** may take a slow path or error. Bun exposes a compatible `process.versions.node` string, but some libraries hardcode version ranges.
- **Node-specific internals** — `process.binding(...)`, some `inspector` hooks, `trace_events` — mostly unimplemented or partial. Rarely hit in practice.

## Running Node-style code under bun

```bash
bun index.js             # just works for most node apps
bun --bun index.js       # force bun runtime even if shebang says #!/usr/bin/env node
```

In bash shebangs, bun also supports `#!/usr/bin/env bun`.

## When in doubt

1. Try under bun first — `bun script.js`.
2. If it breaks, check the module in the compat matrix (link at top).
3. Fall back to node: `node script.js` — node 24.15 is installed on this machine.
4. If reproducible, file a bun issue — the team is responsive.
