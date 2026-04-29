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
grep '"version"' node_modules/mime/package.json | sed 's/.*"version": "\(.*\)".*/\1/'
```
Result: Returns "mime" version `1.6.0` (matches bun.lock)

```bash
grep '"version"' node_modules/postcss-url/node_modules/mime/package.json | sed 's/.*"version": "\(.*\)".*/\1/'
```
Result: Returns "postcss-url" version `2.5.2` (matches bun.lock)

### 2. `--filter a-ui` (correct)

```bash
rm -rf node_modules
bun install --frozen-lockfile --filter a-ui
grep '"version"' node_modules/mime/package.json | sed 's/.*"version": "\(.*\)".*/\1/'
```
Result: Returns "mime" version `1.6.0` (matches bun.lock)

```bash
grep '"version"' node_modules/postcss-url/node_modules/mime/package.json | sed 's/.*"version": "\(.*\)".*/\1/'
```
Result: Returns "postcss-url" version `2.5.2` (matches bun.lock)

### 3. `--filter a-widget` (BUG) 🚨

```bash
rm -rf node_modules
bun install --frozen-lockfile --filter a-widget
grep '"version"' node_modules/mime/package.json | sed 's/.*"version": "\(.*\)".*/\1/'
```
Result: Returns "mime" version `2.5.2` (different from bun.lock) ❌

```bash
grep '"version"' node_modules/postcss-url/node_modules/mime/package.json | sed 's/.*"version": "\(.*\)".*/\1/'
```
Result: No such file

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
