# Codebase Structure

**Analysis Date:** 2026-06-09

## Directory Layout

```
security-reviewer-agent/
├── cinatra/
│   └── oas.json                   # OAS Flow 26.1.0 agent definition (StartNode, ApiNode, EndNode)
├── skills/
│   └── security-review-methodology/
│       └── SKILL.md               # Full LLM prompt: check catalogue, output contract, methodology
├── .github/
│   └── workflows/
│       ├── ci.yml                 # GitHub Actions: build + kind-gates jobs
│       └── release.yml            # GitHub Actions: release pipeline
├── extension-kind-gate.mjs        # Zero-dep CI gate: validates oas.json (agent) or workflow.bpmn (workflow)
├── package.json                   # Package manifest: @cinatra-ai/security-reviewer-agent v0.1.0
├── tsconfig.json                  # TypeScript config (no TS sources currently; present for completeness)
├── .npmrc                         # npm registry config (existence noted; contents not read)
├── LICENSE                        # Apache-2.0
└── README.md                      # Project documentation
```

## Directory Purposes

**`cinatra/`:**
- Purpose: Cinatra platform sidecar directory — the canonical location for agent/workflow artifacts.
- Contains: `oas.json` — the OAS Flow 26.1.0 agent definition consumed by the WayFlow runtime.
- Key files: `cinatra/oas.json`

**`skills/`:**
- Purpose: Skill definitions delivered to the LLM at call time. Each subdirectory is one skill.
- Contains: One skill (`security-review-methodology`) with its `SKILL.md` prompt file.
- Key files: `skills/security-review-methodology/SKILL.md`

**`.github/workflows/`:**
- Purpose: GitHub Actions CI/CD configuration.
- Contains: `ci.yml` (build + kind gate), `release.yml` (release pipeline).
- Key files: `.github/workflows/ci.yml`, `.github/workflows/release.yml`

## Key File Locations

**Entry Points:**
- `cinatra/oas.json`: Runtime entry point — the full flow definition including StartNode, ApiNode, EndNode, edges, and I/O schema.
- `extension-kind-gate.mjs`: CI entry point — `main()` dispatches `runGate()` based on `cinatra.kind` in `package.json`.

**Configuration:**
- `package.json`: Package identity (`@cinatra-ai/security-reviewer-agent`), version (`0.1.0`), license, and Cinatra extension metadata (`cinatra.kind: "agent"`, `cinatra.dependencies: []`).
- `tsconfig.json`: TypeScript compiler configuration.
- `.npmrc`: npm registry settings (existence noted; contents not read).

**Core Logic:**
- `cinatra/oas.json`: All flow logic — node wiring, data flow, LLM call parameters, model preferences.
- `skills/security-review-methodology/SKILL.md`: All reasoning logic — the 8 fuzzy security checks, output contract, non-checks, and step-by-step instructions for the LLM.
- `extension-kind-gate.mjs`: All CI validation logic — `validateAgent`, `validateWorkflow`, `validateBpmnSanity`, `findWorkflowSidecars`, `runGate`.

**Testing:**
- No test files present in this repo. Tests for this agent run in the Cinatra monorepo (skipped in standalone CI when host-internal `@cinatra-ai/*` peers are detected).

## Naming Conventions

**Files:**
- Cinatra sidecar artifacts: lowercase with extension (`oas.json`, `workflow.bpmn`) under `cinatra/`.
- Skills: kebab-case directory name matching the skill identifier (`security-review-methodology`), containing `SKILL.md` (uppercase).
- Gate scripts: kebab-case with `.mjs` extension (`extension-kind-gate.mjs`).
- CI workflows: lowercase kebab-case (`ci.yml`, `release.yml`).

**Directories:**
- Cinatra platform directories: lowercase (`cinatra/`, `skills/`).
- Skill directories: kebab-case matching the skill name (`security-review-methodology/`).

**Package naming:**
- Agent packages: `@cinatra-ai/<slug>-agent` (e.g. `@cinatra-ai/security-reviewer-agent`).
- Workflow packages: `@<scope>/<slug>-workflow` (enforced by `extension-kind-gate.mjs:162`).

**OAS node IDs:**
- Lowercase short identifiers (`start`, `review`, `end`).
- DataFlowEdge names: `<from>_to_<to>_<field>` (e.g. `start_to_review_oasJson`).

## Where to Add New Code

**New security check (fuzzy):**
- Add to the check catalogue in `skills/security-review-methodology/SKILL.md` under "What to check".
- No changes to `cinatra/oas.json` required unless new inputs are needed.

**New flow input:**
- Add to `inputs` array in both the top-level flow and in `$referenced_components.start` inside `cinatra/oas.json`.
- Add a DataFlowEdge from `start` to `review` in `data_flow_connections`.
- Add to the `review` ApiNode's `inputs` and reference via Jinja `{{ var }}` in `data.user`.

**New skill:**
- Create `skills/<skill-name>/SKILL.md` following the existing pattern.
- Register the skill reference in the ApiNode call within `cinatra/oas.json` (platform-specific field — consult Cinatra docs).

**New CI validation rule:**
- Add to the appropriate `validate*` function in `extension-kind-gate.mjs` (e.g. `validateAgent` for agent-kind checks).
- All functions are pure and exported, making them straightforward to extend and test.

**Tests:**
- Tests are not run standalone (monorepo owns them). If adding standalone tests, add a `test` script to `package.json` and place test files at the repo root or a `test/` directory.

## Special Directories

**`cinatra/`:**
- Purpose: Required sidecar directory for Cinatra platform artifacts.
- Generated: No (hand-authored).
- Committed: Yes.

**`skills/`:**
- Purpose: Skill prompt files consumed by the Cinatra LLM bridge at call time.
- Generated: No (hand-authored).
- Committed: Yes.

**`.planning/`:**
- Purpose: GSD planning and codebase map documents (this file's parent).
- Generated: Yes (by GSD tooling).
- Committed: Per project convention.

**`.github/`:**
- Purpose: GitHub Actions CI/CD workflows.
- Generated: No (hand-authored; extraction script appends kind-specific steps to `ci.yml`).
- Committed: Yes.

---

*Structure analysis: 2026-06-09*
