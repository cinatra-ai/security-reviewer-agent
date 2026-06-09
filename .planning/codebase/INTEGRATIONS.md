# External Integrations

**Analysis Date:** 2026-06-09

## APIs & External Services

**Cinatra LLM Bridge:**
- Service: Cinatra internal `/api/llm-bridge` endpoint
  - SDK/Client: HTTP POST via Cinatra `ApiNode` (OAS Flow native); configured in `cinatra/oas.json` under the `review` node
  - Auth: `agent_run_id` actor context header (injected by Cinatra platform at runtime); base URL via `{{CINATRA_BASE_URL}}` platform variable
  - Preferred model: `gpt-5.5` via OpenAI provider (`cinatra_llm.preferredProvider: "openai"`, `cinatra_llm.preferredModel: "gpt-5.5"`)

**OpenAI:**
- Used indirectly through Cinatra's LLM bridge (`preferredProvider: "openai"`); no direct OpenAI SDK or API key in this repo
  - Auth: Managed by Cinatra platform; no credential variables present in this repo

## Data Storage

**Databases:**
- Not applicable — this agent is stateless; it receives an OAS JSON document as input and returns a `findings` JSON array as output

**File Storage:**
- Not applicable

**Caching:**
- Not applicable

## Authentication & Identity

**Auth Provider:**
- Cinatra platform actor envelope — `agent_run_id` is threaded through the flow from `StartNode` → `ApiNode` → `EndNode` (see `cinatra/oas.json` `data_flow_connections`)
  - Implementation: `agent_run_id` passed in the POST body to `/api/llm-bridge` for `enforceRunAccess` enforcement; `riskClass: "read_only"`, `requiresApproval: false`

## Monitoring & Observability

**Error Tracking:**
- Not detected — no Sentry, Datadog, or equivalent SDK

**Logs:**
- Cinatra platform handles execution logging; the agent itself emits only structured JSON output (`{"findings": [...]}`)

## CI/CD & Deployment

**Hosting:**
- Cinatra marketplace — agent published as `@cinatra-ai/security-reviewer-agent` v0.1.0

**CI Pipeline:**
- GitHub Actions — `.github/workflows/ci.yml` (triggers on push/PR to `main`)
  - Node 24, corepack-managed pnpm
  - Validates first-party dep shape (no `@cinatra-ai/*` in deps/devDeps)
  - Runs `extension-kind-gate.mjs` for agent kind: parses `cinatra/oas.json` and scans for retired CRM primitives
  - Skips install/typecheck/test when first-party peers are declared (monorepo owns those steps)
- GitHub Actions — `.github/workflows/release.yml` (release workflow; details not read)

## Environment Configuration

**Required env vars:**
- `CINATRA_BASE_URL` — Cinatra deployment base URL; injected by platform at runtime, referenced as `{{CINATRA_BASE_URL}}` in `cinatra/oas.json`

**Secrets location:**
- No `.env` or secrets files in repo; all credentials and platform tokens managed by Cinatra runtime environment

## Webhooks & Callbacks

**Incoming:**
- Not applicable — this agent is invoked synchronously by the Cinatra OAS Flow engine, not via webhook

**Outgoing:**
- None — the single outgoing call is to Cinatra's internal `/api/llm-bridge` (not an external webhook)

---

*Integration audit: 2026-06-09*
