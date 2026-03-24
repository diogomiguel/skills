---
name: wiremock-isolation
description: Audits and fixes WireMock stub isolation in Playwright E2E test specs. Use when a spec uses WireMock stubs, when investigating concurrency-related flakiness between parallel workers, or when setting up a new spec that creates or modifies WireMock mappings.
argument-hint: [spec-path]
---

# WireMock Stub Isolation

**Target spec:** `$ARGUMENTS`
Strip any leading `@` from the path. Call the cleaned path `SPEC`.

## Context

Multiple Playwright workers share one WireMock server. Any call to `/__admin/mappings/reset`
is a **global wipe** — it destroys custom stubs created by all concurrent workers, causing
404s and race-condition failures in unrelated suites. The canonical fix is:

> Create a `priority: 1` override stub → track its ID → delete only that ID when done.

## Workflow

### 1. Read the spec and its WireMock interactions

Read `SPEC` plus any WireMock stub files it imports (look for `./wiremocks/*.ts`).
Read the `WireMock.ts` utility class to understand available methods.

Identify every place that:
- Calls `WireMock.resetMappings` or `WireMock.resetMappingsInHook`
- Calls `wireMock.updateMapping` without a 404 fallback
- Creates stubs without tracking IDs for targeted cleanup
- Relies on language/data counts that could change with WireMock default updates

### 2. Check against [patterns.md](patterns.md)

Map each finding to a pattern. Prioritise:
1. **P1 – Global reset** (guaranteed to break concurrent workers)
2. **P2 – Update without fallback** (breaks when a concurrent reset wipes the stub)
3. **P3 – Language-count sensitivity** (breaks when WireMock default languages change)
4. **P4 – Heavy serial beforeEach** (performance, not correctness, but degrades CI time badly)

### 3. Apply fixes in priority order

For each finding, apply the fix described in `patterns.md`. Keep changes minimal:
- Do not refactor code unrelated to isolation.
- Do not rename variables or add types unless required by the fix.

After each fix, re-read the affected file to verify the change is correct and the
module-level ID variable (if introduced) is properly scoped.

### 4. Verify no global resets remain

```bash
grep -r "resetMappings\|resetMappingsInHook" tests/
```

Confirm any remaining hits are only inside the `WireMock` class definition itself
(the deprecated static methods), not in any spec or stub helper.

### 5. Check tsconfig if path aliases are introduced

If the fix adds an import that uses a path alias (`@config`, `@resources`, etc.),
check whether the spec file is listed in the `files` array of `tsconfig.json`.
If not, add it — otherwise the alias will fail to resolve at type-check time.

### 6. Smoke-check imports compile

```bash
npx tsc --noEmit
```

Fix any type errors before finishing.

## Reference

- `createMapping` returns the full mapping object including its `id` — always capture it.
- `priority: 1` beats the default priority of `5`. Lower number = higher priority.
- `findMapping` returns mappings sorted by priority — delete any existing override
  *before* calling `findMapping` if you need the unmodified default body.
- Module-level variables in Playwright worker processes are per-worker (separate Node.js
  processes), so tracking IDs at module level is safe across test files on the same worker.
