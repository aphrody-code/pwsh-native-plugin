# Bun as a package manager — full reference

## Lockfile

Bun writes **`bun.lock`** (text, committable). Compatible with npm (`package-lock.json`), yarn v1 (`yarn.lock`), and pnpm (`pnpm-lock.yaml`) on first install — bun migrates them automatically, then you can delete the originals.

- `--frozen-lockfile` — fail if `package.json` drifted from `bun.lock`. Use in CI.
- `--no-save` — install but don't touch `package.json` or lockfile.
- `-y` / `--yarn` — also write a `yarn.lock` v1 (for tools that insist).

## install / add / remove / update

Shared flags across all four:

| Flag | Effect |
| --- | --- |
| `-p` / `--production` | Skip `devDependencies`. |
| `--dry-run` | Report what would happen, don't write. |
| `-f` / `--force` | Re-fetch latest versions regardless of lockfile. |
| `--silent` | No logs. |
| `--verbose` | Chatty logs. |
| `--no-cache` | Ignore manifest cache. |
| `--cache-dir=<path>` | Custom cache path. |
| `--no-verify` | Skip integrity check. Fast, risky. |
| `--ca=<pem>` / `--cafile=<path>` | Custom trusted CA. Needed behind corporate MITM. |
| `-c` / `--config=<path>` | Custom `bunfig.toml`. |
| `--ignore-scripts` | Skip the project's lifecycle scripts (install/uninstall/prepare/etc.). Dep scripts NEVER run unless allowlisted via `trustedDependencies`. |

### add specifics

```bash
bun add express                 # default (dependencies)
bun add -d typescript           # devDependencies
bun add --optional fsevents     # optionalDependencies
bun add --peer react            # peerDependencies
bun add express@^5              # version range
bun add express@latest
bun add lodash@github:lodash/lodash  # GitHub tarball
bun add mypkg@npm:lodash        # rename-install
bun add ./local-pkg             # file: path
bun add 'file:./local-pkg'      # explicit protocol
bun add 'link:../sibling'       # symlink-install (symlink node_modules entry)
bun add 'workspace:*'           # inside a workspace, latest sibling
```

### remove

```bash
bun remove express              # drops from package.json + node_modules
bun rm express lodash           # multiple
```

### update

```bash
bun update                      # everything, within semver
bun update express              # one package
bun update --latest             # break semver, take newest
bun update --latest --workspaces   # monorepo-wide
```

## Auditing and info

```bash
bun audit                       # alias of `bun pm scan`
bun pm scan                     # full vulnerability scan
bun outdated                    # table of deps needing update
bun info react                  # dist-tags, versions, homepage, repo
bun why lyra                    # explain why a transitive dep is present
bun pm why lyra                 # same
```

## Workspaces

Root `package.json`:
```json
{
  "name": "root",
  "private": true,
  "workspaces": ["packages/*", "apps/*"]
}
```

Operations:
```bash
bun install                                  # installs & hoists everything
bun add react -w apps/web                    # add to one workspace
bun remove lodash --workspaces               # remove from all
bun run --workspaces build                   # run script in every workspace that has it
bun run -F 'apps/*' --parallel dev           # filter + parallel
bun run -F 'packages/ui...' build            # "ui and everything that depends on it"
bun outdated --workspaces
bun pm version minor --workspaces            # bump versions across the repo
```

Dep protocols useful in workspaces:
- `"my-ui": "workspace:*"` — always the sibling's current version.
- `"my-ui": "workspace:^"` — sibling, caret-pinned at publish time.
- `"my-ui": "workspace:~"` — sibling, tilde-pinned.

## Catalogs (shared version pins across a monorepo)

`package.json` root:
```json
{
  "workspaces": {
    "packages": ["packages/*"],
    "catalog": {
      "react": "^19.0.0",
      "react-dom": "^19.0.0"
    }
  }
}
```

Then in a package: `"react": "catalog:"` picks from the shared catalog. Update once, propagate everywhere.

## Overrides

```json
{
  "overrides": { "semver": "^7.5.0" },
  "resolutions": { "braces": "^3.0.3" }
}
```

`overrides` is the npm-compatible field; `resolutions` is the yarn one. Both are honored.

## `trustedDependencies` — allowlist postinstall scripts

By default, bun does NOT run dep lifecycle scripts (good for security). To opt in:
```json
{ "trustedDependencies": ["esbuild", "sharp"] }
```

## Patching a dependency

```bash
bun patch lodash              # copies into patches/ for editing
# ...edit files under node_modules/lodash
bun patch --commit lodash     # creates a .patch file and pins it
```

## pm utilities

```bash
bun pm bin                    # project bin folder
bun pm bin -g                 # global bin folder
bun pm list                   # shallow dep tree
bun list                      # alias
bun pm list --all             # full tree
bun pm pack                   # tarball of current package
bun pm pack --destination ./out --filename pkg.tgz
bun pm version patch          # bump version + git tag
bun pm whoami                 # npm auth identity
bun pm view react             # raw metadata (use `bun info` normally)
bun pm pkg get main           # read a single package.json field
bun pm pkg set scripts.test='bun test'
bun pm pkg delete scripts.lint
```

## Publishing

```bash
bun publish                   # publish to npm registry
bun publish --dry-run         # test
bun publish --access public   # for scoped packages
```

Auth: `bun pm whoami` then `npm login` if needed — bun reads `~/.npmrc` for auth tokens.
