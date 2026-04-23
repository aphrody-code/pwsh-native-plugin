# Bun bundler reference — `bun build`

Fast, zero-config bundler for browsers, Node, and Bun targets. Replaces esbuild/webpack/rollup/tsup for most needs.

## Core usage

```bash
bun build ./src/index.ts --outfile dist/index.js
bun build ./src/a.ts ./src/b.ts --outdir dist   # multiple entries
bun build ./src/index.ts --outdir dist --splitting   # code-split shared chunks
bun build ./src/index.ts --target browser --outdir dist
bun build ./src/index.ts --target node    --outdir dist
bun build ./src/index.ts --target bun     --outdir dist   # default
```

## Standalone executables — `--compile`

Bundles your code + Bun runtime into one exe:

```bash
bun build ./cli.ts --compile --outfile mycli.exe
bun build ./cli.ts --compile --production --outfile mycli.exe   # minified
bun build ./cli.ts --compile --bytecode --outfile mycli.exe     # faster cold start
```

Cross-compilation:
```bash
bun build ./cli.ts --compile \
  --target=bun-linux-x64 \      # or bun-darwin-arm64, bun-windows-x64, etc.
  --outfile mycli-linux
```

Standalone-exe runtime autoload flags:
- `--compile-autoload-dotenv` (default true) — load `.env` at runtime.
- `--compile-autoload-bunfig` (default true).
- `--compile-autoload-tsconfig` (default false).
- `--compile-autoload-package-json` (default false).
- `--compile-exec-argv=<args>` — prepend args to the exe's execArgv.

## Common flags

| Flag | Effect |
| --- | --- |
| `--production` | Sets `NODE_ENV=production`, enables minification. |
| `--minify` | Force minify (implied by `--production`). |
| `--minify-whitespace` / `--minify-syntax` / `--minify-identifiers` | Fine-grained. |
| `--sourcemap` | `"inline"`, `"external"`, `"linked"`. |
| `--splitting` | Emit shared chunks. Requires `--outdir`. |
| `--format` | `"esm"` (default), `"cjs"`, `"iife"`. |
| `--external <pkg>` (repeatable) | Don't bundle this dep. Useful for peer deps, node built-ins. |
| `--packages external` | Mark all `package.json` deps as external. Good for libraries. |
| `--define KEY=value` | Replace `KEY` with `value` at build time. |
| `--loader <ext>:<loader>` | e.g. `--loader .svg:text`. Loaders: text, file, js, ts, jsx, tsx, json, toml, wasm, napi. |
| `--public-path <url>` | Prefix for emitted asset URLs. |
| `--conditions <name>` | Package.json export conditions. |
| `--drop console,debugger` | Remove calls at build time. |
| `--watch` | Rebuild on file change. |
| `--metafile <path>` | JSON with module graph, sizes, imports. |
| `--metafile-md <path>` | LLM-friendly markdown module graph. |
| `--bytecode` | Emit bytecode cache. Faster cold start. Implies `--target bun`. |
| `--banner '<text>'` / `--footer '<text>'` | Prepend/append raw text to output. |

## Programmatic API

```ts
const result = await Bun.build({
  entrypoints: ['./src/index.ts'],
  outdir: './dist',
  target: 'browser',
  minify: true,
  sourcemap: 'linked',
  plugins: [myPlugin],
  define: { 'process.env.NODE_ENV': '"production"' },
  external: ['react'],
});
if (!result.success) for (const m of result.logs) console.error(m);
```

## Plugin API (lightweight, esbuild-inspired)

```ts
import type { BunPlugin } from 'bun';

const yamlPlugin: BunPlugin = {
  name: 'yaml-loader',
  setup(build) {
    build.onResolve({ filter: /\.ya?ml$/ }, args => ({ path: args.path, namespace: 'yaml' }));
    build.onLoad({ filter: /\.ya?ml$/, namespace: 'yaml' }, async args => {
      const text = await Bun.file(args.path).text();
      const { default: yaml } = await import('js-yaml');
      return { contents: `export default ${JSON.stringify(yaml.load(text))}`, loader: 'js' };
    });
  },
};
```

Plugins can also be registered globally for `bun run` via `Bun.plugin()`.

## `bunfig.toml` — project config

```toml
[install]
registry = "https://registry.npmjs.org"
cache = "~/.bun/install/cache"
optional = true
dev = true
peer = true
production = false
frozen-lockfile = false
save-text-lockfile = true

[install.scopes]
# Scoped registry (e.g. private packages)
"@my-org" = { url = "https://npm.pkg.github.com", token = "$GITHUB_TOKEN" }

[run]
shell = "bun"        # or "system"
silent = false
bun = false          # if true, scripts use bun instead of node

[test]
root = "./"
coverage = false
coverageSkipTestFiles = true

[bundle]
# defaults for bun build; CLI flags override
```
