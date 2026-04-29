# Bun `--filter` Lockfile Bug

`bun install --filter` doesn't respect the hoisting structure defined in `bun.lock`. It re-computes hoisting based only on the filtered workspace, causing nested dependencies to be placed at the wrong level.

## Project Structure

```
bun-lock-filter/
├── apps/
│   └── a-ui/           → depends on send@0.17.1 (which uses mime@1.6.0)
│       └── package.json   and a-widget (workspace)
├── libs/
│   └── a-widget/       → depends on postcss-url@10.1.3 (which uses mime@2.5.2)
│       └── package.json
├── bun.lock
└── package.json
```

Two different versions of `mime` exist:
- `mime@1.6.0` — used by `send` (dependency of `a-ui`)
- `mime@2.5.2` — used by `postcss-url` (dependency of `a-widget`)

## What `bun.lock` defines

```
"mime":              mime@1.6.0   ← hoisted to root (send uses this)
"postcss-url/mime":  mime@2.5.2   ← nested under postcss-url
```

## Test Results (bun v1.3.11)

### 1. Full install (correct)

```bash
rm -rf node_modules
bun install --frozen-lockfile
```

| package | version | location |
|---------|---------|----------|
| mime | 1.6.0 | root (hoisted, for send) |
| mime | 2.5.2 | nested under postcss-url |

Exit code: **0**. Matches `bun.lock`.

### 2. `--filter a-ui` (correct)

```bash
rm -rf node_modules
bun install --frozen-lockfile --filter a-ui
```

| package | version | location |
|---------|---------|----------|
| mime | 1.6.0 | root (hoisted, for send) |
| mime | 2.5.2 | nested under postcss-url |

Exit code: **0**. Matches `bun.lock`. (a-ui depends on a-widget, so both trees are installed)

### 3. `--filter a-widget` (BUG)

```bash
rm -rf node_modules
bun install --frozen-lockfile --filter a-widget
```

| package | version | location |
|---------|---------|----------|
| mime | **2.5.2** | root |
| mime@1.6.0 | **missing** | — |

Exit code: **0** (should have failed or installed correctly).

`--filter a-widget` only sees `postcss-url` → `mime@2.5.2`, so it hoists `2.5.2` to root. It doesn't know that the full lockfile reserves root `mime` for `1.6.0`.

### 4. `--filter a-widget` without `--frozen-lockfile`

```bash
rm -rf node_modules
bun install --filter a-widget
```

Same result as test 3. `bun.lock` is **not modified** (still contains both `mime@1.6.0` and `postcss-url/mime@2.5.2`), but the installed tree doesn't match.

## Bug Summary

`--filter` re-computes the hoisting layout using only the filtered workspace's dependency tree, ignoring the global hoisting structure recorded in `bun.lock`.

This means:
- Root `node_modules` gets a **different version** than what `bun.lock` specifies
- `--frozen-lockfile` does **not** catch this inconsistency (exit 0)
- Switching between `--filter a-ui` and `--filter a-widget` produces **different root `node_modules`** layouts

## Expected Behavior

`--filter a-widget` should either:
1. Install the same hoisting structure as the full install (root `mime@1.6.0`, nested `postcss-url/node_modules/mime@2.5.2`), or
2. Fail with `--frozen-lockfile` if it can't reproduce the correct structure

## Reproduction

```bash
git clone <this-repo>
cd bun-lock-filter

# correct
rm -rf node_modules
bun install --frozen-lockfile
# check: root mime = 1.6.0, postcss-url/mime = 2.5.2

# bug
rm -rf node_modules
bun install --frozen-lockfile --filter a-widget
# check: root mime = 2.5.2 (wrong!)
```

## Tested Versions

- bun 1.2.15 — **bug exists**
- bun 1.3.11 — **bug exists**
