<!-- refreshed: 2026-06-09 -->
# Architecture

**Analysis Date:** 2026-06-09

## System Overview

```text
┌──────────────────────────────────────────────────────────────┐
│                Cinatra Platform (WayFlow Runtime)             │
│  Caller invokes agent with: oasJson, packageSlug,            │
│  reviewContext, agent_run_id                                  │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│  StartNode ("Inputs")                                        │
│  `cinatra/oas.json` → $referenced_components.start           │
│  Required: oasJson | Hidden: agent_run_id, packageSlug,      │
│  reviewContext                                               │
└────────────────────────┬─────────────────────────────────────┘
                         │ ControlFlowEdge: start_to_review
                         │ DataFlowEdges: all 4 inputs forwarded
                         ▼
┌──────────────────────────────────────────────────────────────┐
│  ApiNode ("Run advisory security review")                    │
│  `cinatra/oas.json` → $referenced_components.review          │
│  POST {{CINATRA_BASE_URL}}/api/llm-bridge                    │
│  Model: openai / gpt-5.5                                     │
│  Skill: security-review-methodology                          │
│  Output key: findings (JSON string)                          │
└────────────────────────┬─────────────────────────────────────┘
                         │ ControlFlowEdge: review_to_end
                         │ DataFlowEdge: findings forwarded
                         ▼
┌──────────────────────────────────────────────────────────────┐
│  EndNode                                                     │
│  `cinatra/oas.json` → $referenced_components.end             │
│  Output: findings (string)                                   │
└──────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| StartNode (start) | Declares and validates flow inputs; marks `oasJson` as required | `cinatra/oas.json` |
| ApiNode (review) | Calls the Cinatra LLM bridge with the security-review prompt; threads `agent_run_id` for actor-envelope enforcement | `cinatra/oas.json` |
| EndNode (end) | Exposes `findings` as the flow output | `cinatra/oas.json` |
| security-review-methodology skill | Provides the full system prompt, check catalogue, and output contract to the LLM | `skills/security-review-methodology/SKILL.md` |
| extension-kind-gate | Self-contained CI gate; validates `cinatra/oas.json` for retired CRM primitives (agent kind) and `cinatra/workflow.bpmn` shape (workflow kind) | `extension-kind-gate.mjs` |

## Pattern Overview

**Overall:** Declarative OAS Flow agent — a three-node linear pipeline (Start → ApiNode → End) backed by a skill-delivered system prompt. No custom TypeScript runtime logic; all reasoning is delegated to the LLM bridge.

**Key Characteristics:**
- Single-pass, stateless: inputs arrive at StartNode, one LLM call is made, findings are returned — no branching, no loops, no HITL gate.
- Skill-as-prompt: the security review methodology is fully encoded in `skills/security-review-methodology/SKILL.md` and injected at LLM call time; the OAS `system` field is a short anchor instruction.
- Advisory-only output: severity values are limited to `"warning"` and `"suggestion"`; the platform's deterministic lint owns `"blocker"`.
- Actor-envelope threading: `agent_run_id` is explicitly forwarded from StartNode through the ApiNode body, satisfying the `enforceRunAccess` requirement.
- Pure inspection: the skill makes no MCP calls, performs no mutations, and produces a single `{"findings":[...]}` JSON object.

## Layers

**Flow Definition Layer:**
- Purpose: Declares the agent's nodes, control flow, data flow, and I/O schema in a platform-readable OAS format.
- Location: `cinatra/oas.json`
- Contains: StartNode, ApiNode, EndNode, ControlFlowEdges, DataFlowEdges, input/output schemas.
- Depends on: Cinatra WayFlow Runtime (OAS Flow 26.1.0 spec).
- Used by: The Cinatra platform at publish/install and at runtime invocation.

**Skill / Prompt Layer:**
- Purpose: Encodes the security-review methodology — what to check, what to ignore, output contract, and step-by-step reasoning instructions — delivered to the LLM as context.
- Location: `skills/security-review-methodology/SKILL.md`
- Contains: System prompt rules, check catalogue (8 fuzzy check types), output JSON shape, and explicit non-checks.
- Depends on: Nothing (pure text consumed by the LLM bridge).
- Used by: The ApiNode's LLM call at runtime.

**CI Gate Layer:**
- Purpose: Lightweight, zero-dependency pre-publish sanity check run in GitHub Actions without registry access.
- Location: `extension-kind-gate.mjs`
- Contains: `validateAgent`, `validateWorkflow`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `findWorkflowSidecars`, `runGate` (all pure functions; exported for testability).
- Depends on: Node.js built-ins only (`fs`, `path`).
- Used by: `.github/workflows/ci.yml` (`kind-gates` job).

## Data Flow

### Primary Request Path

1. **Invocation** — caller supplies `oasJson` (required), plus optional `packageSlug`, `reviewContext`, `agent_run_id` to the StartNode (`cinatra/oas.json` → `$referenced_components.start`).
2. **Input forwarding** — four DataFlowEdges carry all inputs to the ApiNode (`cinatra/oas.json` → `data_flow_connections`).
3. **LLM call** — ApiNode POSTs to `{{CINATRA_BASE_URL}}/api/llm-bridge` with `system` anchor, `user` template (Jinja-interpolated), `agent_id: "security-reviewer-agent"`, `agent_run_id`, and model preference `openai/gpt-5.5` (`cinatra/oas.json` → `$referenced_components.review`).
4. **Skill injection** — WayFlow injects `skills/security-review-methodology/SKILL.md` as additional LLM context at call time.
5. **LLM response** — bridge returns JSON; DataFlowEdge extracts the `findings` key from the response object (`cinatra/oas.json` → `data_flow_connections` → `review_to_end_findings`).
6. **Output** — EndNode surfaces `findings` as the flow output string (`cinatra/oas.json` → `$referenced_components.end`).

### CI Gate Path

1. GitHub Actions `kind-gates` job checks out repo and runs `node extension-kind-gate.mjs --package-root .` (`.github/workflows/ci.yml`).
2. `runGate` reads `package.json`, detects `cinatra.kind === "agent"`, delegates to `validateAgent` (`extension-kind-gate.mjs:352`).
3. `validateAgent` parses `cinatra/oas.json` and walks all LLM-visible fields (`system`, `user`, `description`) looking for banned CRM primitives (`extension-kind-gate.mjs:131`).
4. Violations are printed to stderr; exit code 1 fails the CI job.

**State Management:**
- No persistent state. The agent is fully stateless — each invocation is independent.

## Key Abstractions

**ReviewFinding:**
- Purpose: The canonical output unit — a single security concern with code, severity, message, optional location, and source.
- Examples: Defined in `skills/security-review-methodology/SKILL.md` (output contract section).
- Pattern: Plain JSON object; severity restricted to `"warning"` or `"suggestion"`.

**`{"findings":[...]}` Envelope:**
- Purpose: Required wrapper object. The WayFlow DataFlowEdge extracts the `findings` key by name; a bare array causes a `Cannot index array with string "findings"` runtime error.
- Examples: `skills/security-review-methodology/SKILL.md` (step 1, step 4, Critical note).
- Pattern: Always `{"findings": [...]}` — never a bare `[...]`.

**Fuzzy vs. Deterministic Split:**
- Purpose: Separates LLM-based advisory checks (this agent) from byte-level deterministic lint (monorepo-side scanners).
- Pattern: This agent emits only `"warning"` / `"suggestion"`; deterministic lint owns `"blocker"`. The platform downgrades any helper "blocker" to "warning".

## Entry Points

**Runtime Entry Point:**
- Location: `cinatra/oas.json` → `start_node.$component_ref: "start"`
- Triggers: WayFlow invocation from the Cinatra platform (e.g. the `/chat` agent authoring flow calling this as a sub-agent).
- Responsibilities: Receives inputs, gates on required `oasJson`, routes to ApiNode.

**CI Entry Point:**
- Location: `extension-kind-gate.mjs` → `main()` (invoked when `process.argv[1]` matches the file)
- Triggers: `node extension-kind-gate.mjs --package-root .` in `.github/workflows/ci.yml`.
- Responsibilities: Parse args, dispatch to `runGate`, print results, exit with code 0 or 1.

## Architectural Constraints

- **Threading:** Single-threaded Node.js; the CI gate is synchronous I/O only. The flow runtime is managed by WayFlow (not this repo).
- **Global state:** None. All functions in `extension-kind-gate.mjs` are pure; no module-level mutable singletons.
- **Circular imports:** None — `extension-kind-gate.mjs` imports only Node built-ins.
- **No first-party runtime deps:** `package.json` declares `cinatra.dependencies: []`. The gate script must remain zero-dependency so CI passes before the `@cinatra-ai` registry is reachable.
- **Output envelope:** The LLM MUST return `{"findings":[...]}` — never a bare array. This is a hard constraint enforced by the WayFlow DataFlowEdge extraction.

## Anti-Patterns

### Bare array LLM response

**What happens:** LLM returns `[{...}, {...}]` instead of `{"findings":[...]}`.
**Why it's wrong:** WayFlow's DataFlowEdge tries to index the response with the string key `"findings"`, which fails with `Cannot index array with string "findings"`.
**Do this instead:** Always wrap findings in the object envelope: `{"findings":[...]}`. This applies even for empty results (`{"findings":[]}`) and error fallbacks. See `skills/security-review-methodology/SKILL.md` steps 1 and 4.

### Emitting severity "blocker"

**What happens:** The LLM assigns `severity: "blocker"` to a finding.
**Why it's wrong:** Blocker severity is reserved for the deterministic lint. The platform handler downgrades any advisory "blocker" to "warning", making the output misleading.
**Do this instead:** Use only `"warning"` or `"suggestion"`. See `skills/security-review-methodology/SKILL.md` output contract section.

## Error Handling

**Strategy:** Graceful degradation — if the OAS JSON fails to parse, return a structured finding (`unparseable_oas`) rather than crashing. The envelope contract is always maintained.

**Patterns:**
- Parse failure → return `{"findings":[{"code":"unparseable_oas","severity":"suggestion",...}]}` (skill step 1).
- Nothing found → return `{"findings":[]}` (skill step 4).
- CI gate parse failure → `errors.push(...)` + exit 1 (`extension-kind-gate.mjs:139`).

## Cross-Cutting Concerns

**Logging:** Not applicable at runtime (LLM call). CI gate uses `console.log` / `console.error` for pass/fail output (`extension-kind-gate.mjs:369-378`).
**Validation:** Input validation is minimal — `oasJson` is declared required at StartNode level; the LLM handles malformed JSON gracefully per the skill's step 1 instruction.
**Authentication:** Actor-envelope threading via `agent_run_id` threaded through the ApiNode body satisfies the platform's `enforceRunAccess` requirement.

---

*Architecture analysis: 2026-06-09*
