# Coding Conventions

**Analysis Date:** 2026-06-09

## Naming Patterns

**Files:**
- `kebab-case` for script files: `extension-kind-gate.mjs`
- `camelCase` for exported functions: `parseArgs`, `validateAgent`, `validateWorkflow`, `runGate`, `walkLlmStrings`, `scanOasString`
- `SCREAMING_SNAKE_CASE` for module-level constants: `LLM_VISIBLE_FIELDS`, `BANNED_PRIMITIVES`, `BANNED_TYPEHINTS`, `BPMN_MODEL_NS`
- `PascalCase` suffixed with `_RE` for regex constants: `OBJECTS_LIST_CRM_RE`, `WORKFLOW_PACKAGE_NAME_RE`

**Functions:**
- Verbs first: `validate*` for functions returning `string[]` errors, `scan*` for inspection routines, `walk*` for recursive traversal, `find*` for search/collection, `run*` for dispatch entry points

**Variables:**
- `camelCase` throughout
- Local regex variables use suffix `Re` (lowercase): `tagRe`, `nsRe`, `nm`

**Types / Interfaces:**
- No TypeScript type declarations in this repo — the single implementation file (`extension-kind-gate.mjs`) is plain ESM JavaScript. `tsconfig.json` references a `src/` directory that does not exist; the config is a template placeholder for the extracted-repo pattern.

## Code Style

**Formatting:**
- No formatter config detected (no `.prettierrc`, `biome.json`, or `eslint.config.*`). Code follows a consistent manual style: 2-space indentation, trailing commas in multi-line arrays/objects, single quotes for strings.

**Linting:**
- No ESLint config detected. CI does not run a lint step.

## Import Organization

**Order (observed in `extension-kind-gate.mjs`):**
1. Node built-in modules only — `node:fs`, `node:path`
2. No third-party imports (zero-dependency by design)
3. No project-relative imports (single-file module)

**Module System:**
- ESM (`"type": "module"` in `package.json`), `.mjs` extension for the gate script
- Imports use `node:` protocol prefix: `import { readFileSync, existsSync, readdirSync } from "node:fs"`

**Path Aliases:**
- Not applicable — single-file module, no bundler.

## Error Handling

**Patterns:**
- All public functions return `string[]` errors (never throw). Callers accumulate and check `errors.length`.
- File I/O is wrapped in `try/catch`; errors are pushed as strings: `errors.push(\`could not read package.json: ${err instanceof Error ? err.message : String(err)}\`)`.
- The `main()` function is the only caller that throws to the process — caught at top-level with `process.exit(1)`.
- Early-return pattern when a fatal precondition fails: `if (!existsSync(oasPath)) return errors;`
- Regex parse walks return immediately on first structural error (malformed XML tag balance): `errors.push(...); return errors;`

**Error Message Style:**
- Messages are lowercase sentences, no period at end: `"cinatra/workflow.bpmn is empty"`
- Include the offending value in backtick-JSON form: `got ${JSON.stringify(pkg?.name)}`

## Logging

**Framework:** `console.log` / `console.error` — Node built-ins only, used exclusively in `main()`.

**Patterns:**
- Success: `console.log("✓ extension-kind-gate: ...")` to stdout
- Failures: `console.error("✗ extension-kind-gate: ...")` to stderr, one bullet per error
- Library functions (`validateAgent`, `validateWorkflow`, etc.) are pure — no logging, only return values

## Comments

**When to Comment:**
- Block comments at the top of each logical section using `// ---` separator lines
- Inline `//` comments explain non-obvious decisions: why a check exists, which monorepo rule it mirrors, and what a failing case looks like
- `//` on `package.json` keys (`"//": "..."`) used for JSON comments in `tsconfig.json`

**JSDoc/TSDoc:**
- Not used. Functions have inline `/** ... */` doc comments describing return type and purity: `/** Validate an agent extension at packageRoot. Pure: returns string[] errors. */`

## Function Design

**Size:** Functions are small and focused — largest is `validateBpmnSanity` (~80 lines) which handles a bounded XML walk. All others are under 30 lines.

**Parameters:** Functions accept a single `packageRoot: string` path or a plain value (`xml: string`, `pkg: object`). No option objects. No callbacks except the `onString` callback in `walkLlmStrings`.

**Return Values:**
- Pure validation functions return `string[]` (empty = pass, non-empty = failures)
- `runGate` returns `{ kind, errors }` — a plain object
- `main()` calls `process.exit()` — no return value

## Module Design

**Exports:** Named exports for every testable unit (`parseArgs`, `validateAgent`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `findWorkflowSidecars`, `validateWorkflow`, `runGate`). The `main()` function is NOT exported.

**Barrel Files:** Not applicable — single-file module.

**Self-invocation guard:** Uses `import.meta.url` comparison to detect direct invocation:
```js
const invokedDirectly =
  process.argv[1] && resolve(process.argv[1]) === resolve(new URL(import.meta.url).pathname);
if (invokedDirectly) { ... }
```
This makes the module importable as a library without side effects.

## Cinatra Agent / OAS Conventions

**Skill files** (`skills/*/SKILL.md`) follow YAML front-matter with keys: `name`, `description`, `match_when`.

**OAS output contract:** Agent LLM responses MUST return `{"findings": [...]}` (wrapped object), never a bare array. This is enforced by the skill prompt and documented inline with the reason (`DataFlowEdge` extracts the `findings` key).

**Template interpolation:** Jinja-style `{{ var }}` in `system`/`user` OAS fields. Untrusted inputs (`oasJson`, `packageSlug`, `reviewContext`) flow through `{{ ... }}` in the `user` field only — not in `system`.

---

*Convention analysis: 2026-06-09*
