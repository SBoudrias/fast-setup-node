# fast-setup-node

Fast Node.js setup for GitHub Actions supporting **Yarn 4**, **npm**, and **pnpm**. Caches the linked `node_modules` tree and skips dependency installation completely on an exact cache hit.

## Why

Standard `actions/setup-node` caching only caches package manager download archives (`.yarn/cache`, `~/.npm`, or the pnpm store). Every CI job still starts the package manager, reads the lockfile, unpacks archives, and links thousands of files into `node_modules`.

`fast-setup-node` adds an exact-hit `node_modules` layer in front of your package manager:

1. **Auto-detected package manager:** Detects `yarn`, `npm`, or `pnpm` automatically from `package.json`'s `packageManager` field and lockfiles (`yarn.lock`, `package-lock.json` / `npm-shrinkwrap.json`, `pnpm-lock.yaml`), or accepts an explicit `package-manager` input.
2. **Exact-hit skip-install:** Looks up the linked `node_modules` tree (root + workspace `node_modules`, plus `.yarn/install-state.gz` for Yarn) keyed on package manager, OS, architecture, lockfile, config files, and every workspace `package.json`. On a hit, install is skipped completely.
3. **No stale-tree reconciliation:** Intentionally omits `restore-keys` on `node_modules`. Reconciling a stale `node_modules` tree is slower than doing a clean install from the package manager download cache.
4. **Clean fallback on miss:** On a cache miss, restores the package manager's download cache (auto-detected via `yarn config get cacheFolder`, `npm config get cache`, or `pnpm store path`), runs an immutable/frozen install (`yarn install --immutable`, `npm ci --prefer-offline --no-audit --no-fund`, or `pnpm install --frozen-lockfile --prefer-offline`), and saves both caches immediately—before test or build steps dirty `node_modules` with `.vite`/`.cache` files—so cache-hit jobs never pay post-job save overhead.
5. **Faster Yarn PR installs without weakening lockfile checks:** Yarn enables hardened mode on public pull requests, re-verifying every package resolution against the registry (+3–7 s). When `HEAD` is a PR merge commit (`fetch-depth: 2`) and `yarn.lock` and `.yarnrc.yml` are unchanged from the base commit (`HEAD^1`), hardened mode is disabled while `--immutable` still rejects any `package.json` drift.
6. **Direct binary execution on `PATH`:** Appends `node_modules/.bin` to `GITHUB_PATH` so subsequent steps can run `vitest`, `eslint`, `tsc`, `turbo`, etc. directly without package manager CLI startup overhead. Corepack shims (`yarn` / `pnpm`) are only enabled on cache misses unless `enable-package-manager: 'true'` is requested (`npm` is always available via Node.js).
7. **Node ≥ 25 Corepack compatibility:** Enables the runner's system Corepack shim before `actions/setup-node` switches Node versions, so `yarn` and `pnpm` still work when installing Node versions that no longer bundle Corepack.

### Measured impact

Measured on [Inquirer.js](https://github.com/SBoudrias/Inquirer.js) (a 28-package Yarn 4 monorepo running a 12-job CI matrix across Ubuntu and Windows):

| Metric | Before | After | Change |
| --- | ---: | ---: | ---: |
| CI wall time | 61 s | 34–39 s | -36% to -44% |
| Total runner time (12-job matrix) | 444 s | 189 s | -57% |
| Dependency install on warm hit | runs every job | skipped (0 s) | -100% |

*The end-to-end wall and runner-time reductions above reflect the full CI workflow using this setup pattern (warm `node_modules` skip-install, direct binary execution via `node_modules/.bin` on `PATH`, and parallel job execution).*

## Supported package managers

| Package manager | Lockfile | Config / workspace files hashed | Cold install command | Download cache |
| --- | --- | --- | --- | --- |
| **Yarn 4** (`nodeLinker: node-modules`) | `yarn.lock` | `.yarnrc.yml`, `**/package.json` | `yarn install --immutable` | `yarn config get cacheFolder` |
| **npm** | `package-lock.json` or `npm-shrinkwrap.json` | `.npmrc`, `**/package.json` | `npm ci --prefer-offline --no-audit --no-fund` | `npm config get cache` |
| **pnpm** | `pnpm-lock.yaml` | `pnpm-workspace.yaml`, `.npmrc`, `**/package.json` | `pnpm install --frozen-lockfile --prefer-offline` | `pnpm store path` |

## Usage

### Basic (single package or monorepo)

Works zero-config for Yarn 4, npm, and pnpm across both single-package repositories and monorepos:

```yaml
steps:
  - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
    with:
      persist-credentials: false
      fetch-depth: 2 # enables Yarn PR hardened-mode skip when yarn.lock is unchanged

  - uses: SBoudrias/fast-setup-node@v1
    with:
      node-version: '24'

  # Binaries from node_modules/.bin are directly on PATH
  - run: vitest --run
```

> **Tip:** For supply-chain hardening (and tools like `zizmor`), pin to a full commit SHA with a `# v1` comment rather than a moving tag.

### When a job needs `yarn` or `pnpm` on a cache hit

By default (`enable-package-manager: 'false'`), Corepack shims for `yarn` and `pnpm` are skipped on a `node_modules` cache hit to save setup time on jobs that call binaries directly (like `vitest`, `jest`, `oxlint`). If subsequent steps invoke `yarn` or `pnpm` (for example `yarn tsc` or `pnpm -r build`), pass `enable-package-manager: 'true'`:

```yaml
  - uses: SBoudrias/fast-setup-node@v1
    with:
      node-version: '24'
      enable-package-manager: 'true'

  - run: pnpm -r build
```

(`npm` ships with Node.js and is always available regardless of `enable-package-manager`.)

### Scoping caches by Node.js version (native addons)

By default, the `node_modules` cache key includes `package-manager`, `runner.os`, and `runner.arch`—not `node-version`—so a Node version matrix shares one cached tree. That is safe when your repository disables lifecycle scripts or uses N-API prebuilds.

If your dependencies compile Node-ABI-specific native addons during install, scope the cache per Node version with `cache-key-prefix`:

```yaml
  - uses: SBoudrias/fast-setup-node@v1
    with:
      node-version: ${{ matrix.node-version }}
      cache-key-prefix: node-${{ matrix.node-version }}
```

## Monorepo globbing and performance tuning

By default, `fast-setup-node` works with any repository layout without configuration:

- **`cache-paths` default:**
  ```text
  **/node_modules
  !**/node_modules/**/node_modules
  .yarn/install-state.gz   # Yarn only
  ```
  The `!**/node_modules/**/node_modules` exclusion is important: `actions/cache` resolves every matched directory into a tar manifest, so a bare `**/node_modules` glob would match nested dependency directories inside `node_modules/` (including pnpm's `node_modules/.pnpm/*/node_modules`) and archive them twice. Excluding nested `node_modules` caches only the root and workspace `node_modules` directories (each of which still includes its own nested dependencies when archived).
- **`manifest-paths` default:**
  Hashes the detected lockfile, package manager config files (`.yarnrc.yml`, `pnpm-workspace.yaml`, `.npmrc`), and every workspace `package.json` while ignoring `node_modules/`, `.yarn/`, and `.git/`.

On a clean checkout before install, `**/package.json` is fast (~18 ms on a 28-package monorepo). On a cache **miss**, saving `**/node_modules` has to walk `node_modules/` to check for exclusions (~1 s on a ~27k-entry tree). For very large monorepos—or repositories with unrelated `package.json` test fixtures—you can narrow both globs explicitly so globbing stays under ~5 ms:

```yaml
  - uses: SBoudrias/fast-setup-node@v1
    with:
      node-version: '24'
      cache-paths: |
        node_modules
        packages/*/node_modules
        tools/*/node_modules
        .yarn/install-state.gz
      manifest-paths: |
        yarn.lock
        .yarnrc.yml
        package.json
        packages/*/package.json
        tools/*/package.json
```

When set, `cache-paths` and `manifest-paths` are evaluated relative to the repository root (`GITHUB_WORKSPACE`).

## Inputs

| Input | Required | Default | Description |
| --- | :---: | --- | --- |
| `node-version` | No | `'24'` (if `node-version-file` is unset) | Node.js version spec passed to `actions/setup-node`. |
| `node-version-file` | No | `''` | File containing the Node.js version spec (e.g. `.nvmrc`, `.node-version`, `package.json`). |
| `package-manager` | No | `'auto'` | Package manager to use: `'auto'`, `'yarn'`, `'npm'`, or `'pnpm'`. |
| `enable-package-manager` | No | `'false'` | Enable the Corepack shim (`yarn`/`pnpm`) even on a `node_modules` cache hit (`npm` is always available). |
| `enable-yarn` | No | `'false'` | Alias for `enable-package-manager` when using Yarn. |
| `working-directory` | No | `'.'` | Directory containing `package.json` and lockfile. |
| `cache-key-prefix` | No | `''` | Optional prefix prepended to cache keys (e.g. to scope caches by Node version or bust caches). |
| `cache-paths` | No | `**/node_modules` (excluding nested `node_modules`), plus `.yarn/install-state.gz` for Yarn | Multiline glob patterns passed to `actions/cache` for the linked dependency tree. |
| `manifest-paths` | No | Lockfile, config files, and `**/package.json` (excluding `node_modules`, `.yarn`, `.git`) | Multiline glob patterns passed to `hashFiles()` for the `node_modules` cache key. |

## Outputs

| Output | Description |
| --- | --- |
| `cache-hit` | `'true'` if `node_modules` was restored from cache (install skipped); `'false'` if dependencies were installed. |
| `node-version` | The installed Node.js version reported by `actions/setup-node`. |
| `package-manager` | The resolved package manager (`'yarn'`, `'npm'`, or `'pnpm'`). |

## Requirements & security notes

- **Yarn 4 with `nodeLinker: node-modules`:** Repositories using Yarn Plug'n'Play (`.pnp.cjs`) do not create `node_modules` directories and are not supported.
- **Release workflows:** Do not use cached `node_modules` in release or publish workflows. Run a clean `actions/setup-node` (with `package-manager-cache: false`) + immutable install for supply-chain safety.
- **Hardened mode override (Yarn):** Set `YARN_ENABLE_HARDENED_MODE=true` in the job or step environment if you want Yarn hardened mode to always run on pull requests regardless of lockfile diffs.
- **Pinned upstream actions:** All upstream actions (`actions/cache`, `actions/setup-node`) are pinned by full commit SHA, kept up to date by Dependabot, and checked with [zizmor](https://docs.zizmor.sh/).

## License

[MIT](./LICENSE)
