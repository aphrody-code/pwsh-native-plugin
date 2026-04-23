# Bun Shell — `Bun.$` and `bun exec`

Cross-platform POSIX-ish shell written in Zig. Works identically on Windows, macOS, and Linux. Source of truth: <https://bun.com/docs/runtime/shell>.

## `Bun.$` tagged-template API

```ts
import { $ } from 'bun';

// Simple — throws on non-zero exit
await $`ls -la`;
```

### Execution methods (chainable)

| Method | Returns / effect |
| --- | --- |
| `.text()` | `Promise<string>` — stdout as UTF-8 string. Auto-quiets stdout. |
| `.json()` | `Promise<T>` — stdout parsed as JSON. |
| `.blob()` | `Promise<Blob>` — stdout as Blob. |
| `.arrayBuffer()` | `Promise<ArrayBuffer>` |
| `.bytes()` | `Promise<Uint8Array>` |
| `.lines()` | `AsyncIterable<string>` — iterate stdout line by line. |
| `.quiet()` | Suppress stdout/stderr printing (output still captured). |
| `.nothrow()` | Don't throw on non-zero exit — returns the result with `exitCode`, `stdout`, `stderr`. |
| `.throws(boolean)` | Explicit throw toggle (default `true`). |
| `.env({...})` | Override env for this command (does not inherit process.env unless you spread it). |
| `.cwd(path)` | Set cwd for this command. |

Set defaults globally:
```ts
$.env({ CI: 'true' });   // all subsequent commands inherit
$.cwd('./workspace');
```

Direct result (when awaited without `.text()` etc.):
```ts
const r = await $`grep needle haystack`.nothrow();
r.exitCode       // number
r.stdout         // Buffer
r.stderr         // Buffer
r.text()         // helper on the result
```

### Interpolation safety

All `${...}` substitutions are **shell-injection-safe** — each variable becomes a single quoted argument:

```ts
const name = "bobby'; rm -rf /";
await $`mkdir ${name}`;   // mkdir "bobby'; rm -rf /"  -- SAFE
```

Arrays are expanded as multiple arguments:
```ts
const files = ['a.txt', 'b.txt', 'c d.txt'];
await $`rm ${files}`;    // rm a.txt b.txt "c d.txt"
```

Opt out (rare, dangerous) — use `{ raw: '...' }`:
```ts
const flag = Math.random() > 0.5 ? '--verbose' : '--quiet';
await $`ls ${{ raw: flag }}`;   // no escaping
```

Helpers:
```ts
$.escape('hello world');    // 'hello\\ world'
$.braces('a{1,2,3}');       // ['a1','a2','a3']
```

### Redirects (in the template string)

| Syntax | Effect |
| --- | --- |
| `<` | stdin from file / `Bun.file()` / `Buffer` / typed array / `Response` |
| `>` / `1>` | stdout to file / buffer / response |
| `2>` | stderr to file |
| `&>` | both stdout + stderr |
| `>>` / `1>>` | append stdout |
| `2>>` | append stderr |
| `&>>` | append both |
| `2>&1` | merge stderr into stdout |
| `1>&2` | reverse merge |

JS objects on either side:
```ts
const log = Bun.file('./build.log');
await $`tsc -b &> ${log}`;

const input = new TextEncoder().encode('line1\nline2\n');
await $`grep line2 < ${input}`;
```

### Pipes and substitution

```ts
await $`cat file.txt | grep TODO | wc -l`;

const branch = await $`git rev-parse --abbrev-ref HEAD`.text();
await $`gh pr create --head ${branch.trim()}`;

// Command substitution inside the shell string
await $`echo "current branch: $(git rev-parse --abbrev-ref HEAD)"`;
// Note: backticks `` `cmd` `` do NOT work -- use $(...)
```

### Builtins (implemented natively, cross-platform)

`cd`, `pwd`, `echo`, `cat`, `ls` (with `-l`), `rm`, `mkdir`, `touch`, `mv`, `which`, `exit`, `true`, `false`, `yes`, `seq`, `dirname`, `basename`, `bun` (the current bun binary).

Anything else calls out to a real executable on PATH.

### Control flow — use JS, not shell

Bun's shell deliberately does NOT implement `if`/`for`/`while`/`case`/arithmetic/process-substitution/`&` background. Use JS instead:

```ts
// instead of `for f in *.ts; do tsc $f; done`
import { Glob } from 'bun';
for await (const f of new Glob('*.ts').scan()) {
  await $`tsc ${f}`;
}

// instead of `test -f foo.txt && cat foo.txt`
if (await Bun.file('foo.txt').exists()) {
  console.log(await Bun.file('foo.txt').text());
}
```

## `bun exec` — run a shell script file

```bash
bun exec ./build.sh                  # Bun's shell, not bash -- cross-platform
bun exec -e 'git status | wc -l'     # inline
bun build.sh                         # bare .sh -- also works (bun detects shell)
```

## In `package.json` scripts

By default, `bun run <script>` executes the command through Bun's shell — so a script like `"clean": "rm -rf dist"` works on Windows without `rimraf`.

```toml
# bunfig.toml
[run]
shell = "bun"    # default
# shell = "system"   # to use bash / cmd instead
```

Override per-invocation:
```bash
bun run --shell=system build
```

## Writing cross-platform scripts

Prefer `.ts` with `Bun.$` over `.sh` / `.ps1`:

```ts
// scripts/release.ts
import { $ } from 'bun';
const [, , version] = process.argv;
if (!version) throw new Error('usage: bun scripts/release.ts <version>');

await $`bun pm version ${version}`;
await $`git push --follow-tags`;
await $`bun publish --access public`;
```

Run with `bun scripts/release.ts 1.2.3` — identical on Windows / macOS / Linux.

## Security notes

- Variable interpolation is safe by default (auto-quoted).
- `{ raw: '...' }` OPTS OUT — only use with validated/controlled input.
- Spawning `bash -c '...user input...'` reintroduces injection risk — avoid it.
- Argument injection is still possible if the external program parses its own flags from a user-controlled string (e.g., `git log $user_input` where `$user_input` = `--exec=rm`). Validate at the application layer.
