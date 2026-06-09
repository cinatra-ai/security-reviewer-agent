# Testing Patterns

**Analysis Date:** 2026-06-09

## Test Framework

**Runner:**
- Not detected in this repo. No `jest.config.*`, `vitest.config.*`, or test runner dependency in `package.json`.
- The repo is a **source mirror** (it declares host-internal `@cinatra-ai/*` as optional peer dependencies). Per the CI contract in `.github/workflows/ci.yml`, the cinatra monorepo owns running tests for source-mirror repos — standalone test execution is explicitly skipped in CI.

**Assertion Library:**
- Not applicable — no test files present.

**Run Commands:**
```bash
# CI step (standalone repos only — skipped for this source mirror):
corepack pnpm test --if-present

# Kind-gate validation (runs in CI for all repos):
node extension-kind-gate.mjs --package-root .
```

## Test File Organization

**Location:**
- No test files detected anywhere in the repo (`*.test.*`, `*.spec.*` not found).

**Naming:**
- Not applicable.

## Test Structure

**Suite Organization:**
- Not applicable — no test files exist in this repo. The monorepo's test suite covers the exported functions from `extension-kind-gate.mjs`.

## Mocking

**Framework:** Not applicable.

**What to Mock (guidance from function design):**
- `extension-kind-gate.mjs` exports pure functions that accept strings and return `string[]`. These are designed for direct unit testing without mocks: `validateBpmnSanity(xml)`, `validateWorkflowPackageShape(pkg)`, `validateAgent(packageRoot)`.
- File I/O functions (`validateAgent`, `validateWorkflow`, `findWorkflowSidecars`, `runGate`) depend on `readFileSync`/`existsSync`/`readdirSync` from `node:fs`. Tests covering these would mock the filesystem or use a temp directory.

**What NOT to Mock:**
- The pure string-in / string[]-out functions (`validateBpmnSanity`, `validateWorkflowPackageShape`) require no mocking — pass raw strings directly.

## Fixtures and Factories

**Test Data:**
- Not applicable — no fixtures directory exists.
- When adding tests, XML strings and `package.json` objects should be defined inline or in a `test/fixtures/` directory.

## Coverage

**Requirements:** Not enforced — no coverage configuration detected.

**View Coverage:**
```bash
# Not configured. Would require adding a test runner and coverage reporter.
```

## Test Types

**Unit Tests:**
- Intended test targets (exported pure functions in `extension-kind-gate.mjs`):
  - `parseArgs` — arg vector → `{ packageRoot }`
  - `validateWorkflowPackageShape(pkg)` — plain object → `string[]`
  - `validateBpmnSanity(xml)` — XML string → `string[]`
  - `findWorkflowSidecars(packageRoot)` — filesystem walk → `string[]`
  - `validateAgent(packageRoot)` — reads `cinatra/oas.json` → `string[]`
  - `validateWorkflow(packageRoot)` — reads `package.json` + `cinatra/workflow.bpmn` → `string[]`
  - `runGate(packageRoot)` — dispatch → `{ kind, errors }`

**Integration Tests:**
- Not applicable.

**E2E Tests:**
- Not applicable.

## CI Gate (Functional Equivalent of Tests)

The `kind-gates` job in `.github/workflows/ci.yml` exercises the gate against the repo's own `cinatra/oas.json` on every push/PR:

```yaml
- name: Agent OAS validation gate
  run: node extension-kind-gate.mjs --package-root .
```

This acts as a functional smoke test: if `extension-kind-gate.mjs` fails to parse `cinatra/oas.json` or finds banned primitives, CI fails. This is the only automated correctness check that runs standalone (without the monorepo).

## Common Patterns

**Async Testing:**
- Not applicable — all exported functions are synchronous.

**Error Testing:**
- The return-value convention (`string[]`) makes error-case assertions straightforward:
```js
// Example pattern for when tests are added:
const errors = validateBpmnSanity("<not-xml>");
assert(errors.length > 0);
assert(errors[0].includes("no root element"));
```

**Empty-pass assertion:**
```js
const errors = validateWorkflowPackageShape({ name: "@foo/bar-workflow", cinatra: { kind: "workflow", apiVersion: "cinatra.ai/v1", workflowVersion: 1 } });
assert.deepEqual(errors, []);
```

---

*Testing analysis: 2026-06-09*
