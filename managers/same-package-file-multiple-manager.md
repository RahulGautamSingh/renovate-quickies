## How renovate handles multiple managers resolving same package file?

Using packageFile: `pyproject.toml` and managers `pep621`, `pixi` and `poetry` as example

### 1. File-pattern overlap is real and intentional

All three subscribe to pyproject.toml:

| Manager | managerFilePatterns |
| ------- | ------------------- |
| pep621  | `/(^/)pyproject.toml$/`|
| poetry  | `/(^/)pyproject.toml$/`|
| pixi    | `/(^/)pyproject.toml$/` and `/(^|/)pixi.toml$/`|


So Renovate does hand the same file to all three manager's `extractPackageFile`. 

### 2. Each manager bails unless its section exists

The file is parsed three times, but each manager returns `null` if its own table isn't present, so most repos only get one real extraction:

- `poetry`: schema requires `[tool.poetry]`; no `tool.poetry` → `null` (poetry/schema.ts:304, extract.ts:27).
- `pixi`: `getUserPixiConfig` returns `val.tool?.pixi ?? null`; no `[tool.pixi]` → `null` (pixi/extract.ts:38).
- `pep621`: keys off `[project]` / `[build-system]` / `[dependency-groups]` / `[tool.uv|pdm|hatch]`; none present → `null` (pep621/extract.ts:104).

A pixi project (`[tool.pixi]`) and a poetry project (`[tool.poetry]`) are looking at completely different tables, so they never collide. Pixi is fully orthogonal to the other two — no precedence relationship at all.

### 3. The one genuine conflict: pep621 vs poetry

A modern Poetry file can have both a PEP 621 `[project]` table and `[tool.poetry]`. In that situation `pep621` and `poetry` would both extract the same deps from the same file — a true double-handling.

That's resolved by `supersedesManagers`:


```ts
// poetry/index.ts:14
export const supersedesManagers = ['pep621'];
```

Handled in lib/workers/repository/extract/supersedes.ts. After extraction, when `poetry` and `pep621` both produced results for the same `pyproject.toml`:
- if `poetry` has a `lockfile` for it, `poetry` wins and `pep621`'s entry is dropped;
- otherwise the superseded manager (`pep621`) is filtered out for that file.

So `poetry` wins on shared files, and `pep621` only "survives" on files poetry didn't claim.

### Why it never literally collides

Extraction results are namespaced per manager — `extractResult.packageFiles[manager]` (extract/index.ts:93). Each manager keeps its own bucket, so "the same file appearing under three managers" is structurally fine; the supersedes pass is the only place where overlapping content gets deduped.

### TLDR;
parsed by all three, extracted by at most the ones whose table is present, and the only real overlap (`pep621`↔`poetry`) is settled by `supersedesManagers`. `pixi` just minds its own `[tool.pixi]` business.
