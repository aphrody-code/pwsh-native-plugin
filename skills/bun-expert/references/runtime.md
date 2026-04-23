# Bun runtime reference

`bun run` (or just `bun <file>`) executes TypeScript, JSX/TSX, CJS and ESM with no build step. This doc covers the runtime APIs you'll actually reach for.

## HTTP server — `Bun.serve`

```ts
const server = Bun.serve({
  port: 3000,
  // Route-object style (preferred, since 1.1+)
  routes: {
    '/':          () => new Response('hello'),
    '/api/users/:id':  req => Response.json({ id: req.params.id }),
    '/static/*':  new Response(Bun.file('./public/index.html')),
  },
  // Per-method:
  // routes: { '/api': { GET: …, POST: … } },
  fetch(req) { return new Response('fallback'); },
  error(err) { return new Response(String(err), { status: 500 }); },
  idleTimeout: 30,
  development: true,   // nicer errors, HMR in Bun.build integration
  tls: { cert: Bun.file('./cert.pem'), key: Bun.file('./key.pem') },
});
console.log(`Listening on ${server.url}`);
server.reload({ fetch: newHandler });   // hot-swap handler without dropping sockets
```

Upgrade to WebSocket inside a handler:
```ts
Bun.serve({
  fetch(req, server) {
    if (server.upgrade(req, { data: { userId: 1 } })) return;
    return new Response('HTTP');
  },
  websocket: {
    message(ws, msg) { ws.send(msg); },
    open(ws) { ws.subscribe('room-1'); },
    close(ws) { /* cleanup */ },
  },
});
```

## HTTP client — built-in fetch + HTTP/2

`fetch` works out of the box; bun adds:
- `HTTP/2` support via `new Bun.fetch.dialer({ http2: true })` (check current docs).
- `fetch.preconnect('https://example.com')` to warm the connection pool.
- `--fetch-preconnect=<url>` CLI flag.

## Environment variables

Automatic load order (first match wins for conflicts):
1. `.env.{NODE_ENV}.local`
2. `.env.local` (not `test`)
3. `.env.{NODE_ENV}`
4. `.env`

Override with `--env-file=<path>` (repeatable) or disable entirely with `--no-env-file`.

In code:
```ts
Bun.env.DATABASE_URL   // typed as string | undefined (via tsconfig integration)
process.env.FOO        // still works
```

## SQL — `Bun.sql` (Postgres) + `bun:sqlite`

Postgres:
```ts
import { sql } from 'bun';
const users = await sql`SELECT * FROM users WHERE id = ${id}`;
// Template literal protects against injection; parameters are bound.
```

SQLite:
```ts
import { Database } from 'bun:sqlite';
const db = new Database('mydb.sqlite');
const stmt = db.prepare('SELECT * FROM users WHERE id = ?');
stmt.get(42);
stmt.all();
db.transaction(() => { /* ... */ })();
```

## Redis — `Bun.redis`

```ts
import { redis } from 'bun';
await redis.set('k', 'v');
await redis.get('k');
// preconnect with --redis-preconnect or env REDIS_URL
```

## Filesystem — `Bun.file` / `Bun.write`

```ts
const text = await Bun.file('./README.md').text();
const bytes = await Bun.file('./icon.png').arrayBuffer();
await Bun.write('./out.json', JSON.stringify(data, null, 2));
await Bun.write('./copy.bin', Bun.file('./src.bin'));   // efficient copy
```

Streams: `Bun.file(path).stream()` → `ReadableStream`.

## Hashing, crypto, password

```ts
Bun.hash('abc');                        // 64-bit Wyhash
Bun.hash.crc32('abc');
const h = await Bun.password.hash('pw'); // bcrypt/argon2id
await Bun.password.verify('pw', h);
new Bun.CryptoHasher('sha256').update('x').digest('hex');
```

## FFI — `bun:ffi`

Load a native .dll / .so / .dylib:
```ts
import { dlopen, FFIType, suffix } from 'bun:ffi';
const { symbols } = dlopen(`libadd.${suffix}`, {
  add: { args: [FFIType.i32, FFIType.i32], returns: FFIType.i32 },
});
symbols.add(2, 3); // 5
```

For direct C++ embedding: Bun has an experimental C-API (`bun.h`).

## Spawn child processes — `Bun.spawn` / `Bun.spawnSync`

```ts
const proc = Bun.spawn(['git', 'status', '--porcelain'], {
  stdout: 'pipe', stderr: 'pipe',
  cwd: './repo', env: { ...process.env, FORCE_COLOR: '1' },
});
const out = await new Response(proc.stdout).text();
const exit = await proc.exited;
```

Synchronous: `Bun.spawnSync(cmd).stdout.toString()`.

## IPC with workers

`new Worker(new URL('./worker.ts', import.meta.url))` — standard Web Worker API. `SharedArrayBuffer` works.

## Profilers (built-in, no external tooling needed)

- `bun --cpu-prof script.ts` → writes `.cpuprofile` for DevTools.
- `bun --cpu-prof-md script.ts` → writes a grep-friendly markdown table. Pipe into an LLM for analysis.
- `bun --heap-prof script.ts` → `.heapsnapshot`.
- `bun --heap-prof-md script.ts` → markdown heap diff.

## Debugger

```bash
bun --inspect=0.0.0.0:9229 ./server.ts
bun --inspect-brk ./script.ts    # break on line 1
bun --inspect-wait ./script.ts   # wait for connection before executing
```

Connect from Chrome DevTools at `chrome://inspect` (with bun's bridge), VS Code's built-in JS debugger, or WebStorm.
