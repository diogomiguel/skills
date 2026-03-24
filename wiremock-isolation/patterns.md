# WireMock Isolation Patterns

---

## P1 — Global Reset (Critical)

**Symptom**: Tests pass in isolation but fail when run in parallel. Failures are 404s or
unexpected data in stubs that were created earlier in the same suite.

**Root cause**: `WireMock.resetMappings` / `resetMappingsInHook` calls
`/__admin/mappings/reset`, which wipes **all** custom stubs on the shared server —
including those created by concurrent workers.

**Bad**:
```ts
test.afterAll(async ({ browser }) => {
  await WireMock.resetMappingsInHook(browser) // wipes every worker's stubs
})
```

**Good** — create a `priority: 1` override, track its ID, delete only that ID:
```ts
// Module-level per worker process — safe, not shared across workers
let myStubId: string | null = null

export const createMyStub = async (page: Page): Promise<void> => {
  const wireMock = new WireMock(page)
  // Remove own previous stub first so findMapping reads the clean default
  if (myStubId) {
    await wireMock.removeMapping(myStubId)
    myStubId = null
  }
  const defaultMapping = await wireMock.findMapping<MyType>('/api2/my-endpoint', 'GET')
  const created = await wireMock.createMapping<MyType>({
    request: { method: 'GET', urlPathPattern: '/api2/my-endpoint' },
    response: {
      ...defaultMapping.response,
      jsonBody: { /* modified body */ },
    },
    priority: 1, // overrides the default (priority 5) without touching it
  })
  myStubId = created.id
}

export const removeMyStub = async (browser: Browser): Promise<void> => {
  if (myStubId === null) return
  const context = await browser.newContext()
  const page = await context.newPage()
  try {
    await new WireMock(page).removeMapping(myStubId)
    myStubId = null
  } finally {
    await page.close()
    await context.close()
  }
}
```

---

## P2 — Update Without 404 Fallback

**Symptom**: A serial suite fails mid-run with a WireMock 404 on an `updateMapping` call.
Subsequent tests in the suite are skipped.

**Root cause**: A concurrent worker called `resetMappings` between the point where the stub
was created and the point where it was updated, wiping the stub's ID from WireMock.

**Bad**:
```ts
await wireMock.updateMapping({ id: mappingId, ...shape })
// If mappingId no longer exists → 404 → unhandled rejection → test failure
```

**Good** — wrap in try/catch and recreate on 404:
```ts
try {
  await wireMock.updateMapping({ id: mappingId, ...shape })
} catch {
  // Mapping was wiped by a concurrent reset; recreate it.
  await wireMock.createMapping(shape)
}
```

---

## P3 — Language-Count Sensitivity

**Symptom**: Tests asserting segment counts, approval percentages, or reviewer-row counts
fail non-deterministically depending on how many languages WireMock returns.

**Root cause**: WireMock default mappings can evolve (e.g. adding languages to mock data).
Tests that assert an exact count or percentage break whenever the default changes.

**Fix** — pin the language set in `beforeEach` with a language stub:
```ts
import {
  createLanguagesListStub,
  createLanguagesStatisticsStub,
  restoreOriginalMappings,
} from './wiremocks/languagesStub'

test.beforeEach(async ({ page, browser }) => {
  await restoreOriginalMappings(browser) // remove this worker's previous override
  await createLanguagesListStub(page)    // filter /api2/projects/:id/languages → de/en/es
  await createLanguagesStatisticsStub(page) // filter /api2/projects/:id → de/en/es stats
  // ... rest of setup
})
```

**Apply to any spec that**:
- Asserts `verifyNumberOfSegments(N)`
- Asserts an approval/review percentage (`'25% segments approved'`)
- Asserts reviewer-row counts
- Navigates to the editor (even without count assertions — fewer languages = faster load)

**Update hardcoded counts** to match the pinned language set (e.g., 2 target languages):
- `verifyNumberOfSegments(3)` → source (en) + de + es
- `'25% segments approved'` → 1 approved out of 2 targets × 2 segments = 4 total

---

## P4 — Heavy Setup in Serial `beforeEach`

**Symptom**: A serial suite takes N × (full setup time) when it only needs 1 × (full setup
time). E.g. 13 tests × 30 s = 6.5 min instead of 30 s + 13 × 5 s = 95 s.

**Root cause**: `mode: 'serial'` was added to protect shared WireMock state, but the full
project-creation + upload + translate flow was left in `beforeEach` and now runs N times.

**Fix** — run the expensive setup once in `beforeAll`, capture the result (typically the
editor URL), and have `beforeEach` do only fast navigation:

```ts
let savedEditorUrl: string

test.beforeAll(async ({ browser }) => {
  const context = await browser.newContext()
  const page = await context.newPage()
  try {
    await restoreOriginalMappings(browser)
    await createLanguagesListStub(page)
    await createLanguagesStatisticsStub(page)
    await BaseMockTest.setupProjectAndTranslate(page, ...)
    savedEditorUrl = page.url()
  } finally {
    await page.close()
    await context.close()
  }
})

test.afterAll(({ browser }) => restoreOriginalMappings(browser))

test.beforeEach(async ({ page }) => {
  // Fast: mock login (~1 s) + navigate to captured URL (~2 s)
  await page.goto(`${baseConfig.mockPublicApiUrl}/login`)
  await page.waitForLoadState('networkidle')
  await page.evaluate(() => localStorage.setItem('hasSeenExplanationModal', 'true'))
  await page.goto(savedEditorUrl)
})
```

---

## P5 — Flaky Dropdown Interactions

**Symptom**: Clicking a dropdown button succeeds but the menu item is not found or is
hidden before the next click fires. Fails on slow CI / PREnvs on first run.

**Root cause**: Background JS activity (route interception, React re-renders) can close
the dropdown between the button click and the menu-item assertion.

**Fix** — wrap the open + visibility check in `toPass` so the whole sequence retries:
```ts
const deleteMenuItem = page.getByRole('menuitem', { name: 'Delete translation memory' })

await expect(async () => {
  await actionsButton.click()
  await expect(deleteMenuItem).toBeVisible({ timeout: 2000 })
}).toPass({ timeout: TIMEOUTS.DEFAULT })

await deleteMenuItem.click()
```

---

## General Rules

| Rule | Reason |
|------|--------|
| Never call `resetMappings` / `resetMappingsInHook` in specs | Global wipe breaks concurrent workers |
| Always capture the `id` from `createMapping` | Required for targeted cleanup |
| Delete own stub before calling `findMapping` for the default | WireMock returns highest-priority first; your own override hides the default |
| Wrap `updateMapping` in try/catch with `createMapping` fallback | Stubs can be wiped between creation and update by concurrent resets |
| Add language stubs to every spec that opens the editor | Protects against WireMock default data evolving |
| Prefer `beforeAll` for expensive one-time setup in serial suites | Avoids N × setup cost |
