# Security Reviewer Agent

An advisory security-review helper that sits beside the Cinatra chat agent authoring experience. It reads a draft OAS Flow agent definition and surfaces fuzzy risks a deterministic linter cannot see — prompt-injection openings, over-broad tool permissions, missing authorization checks, suspicious external host references, and credential-shaped variables sneaking into prompt fields. Reviews are advisory-only: findings carry severity `warning` or `suggestion`, never `blocker`.

## Works with

- Cinatra chat agent authoring (`/chat` workflow, OAS Flow 26.1.0 format)

## Capabilities

- Accept a draft agent OAS body (`oasJson`, required) plus optional `packageSlug` and `reviewContext` hints; return a `findings` array of `ReviewFinding` objects via the agent output
- Spot prompt-injection openings — untrusted `{{ … }}` interpolations in `system` or `user` fields that lack an explicit trust boundary
- Flag scope-bypass patterns — instructions in prompts that attempt to skip auth checks, ignore prior instructions, or enter admin mode
- Catch credential-shaped Jinja variables (`apiKey`, `token`, `secret`, `bearer`, `password`) interpolated into `system` fields, where the design implies a live credential flowing through the prompt
- Surface suspicious external host references in prompt text (third-party hostnames, raw IPs) that the deterministic URL scan cannot see
- Recommend approval gates when an ApiNode performs a mutating call (`POST`/`PUT`/`DELETE`) to a non-Cinatra URL without a preceding HITL gate
- Detect missing actor envelope threading — a mutating ApiNode calling `/api/llm-bridge` that omits `agent_run_id`, bypassing `enforceRunAccess`
- Flag over-broad MCPToolBox permissions where mutating primitives are exposed but the agent's role only requires read-only access
- Warn on PII (email addresses, phone numbers, account IDs) embedded as literals in `description` or `system` fields
- Return `{"findings":[]}` when no fuzzy risks are detected; fail gracefully with a `suggestion` finding when `oasJson` is not valid JSON
- No credentials or network calls required; the review is a pure prompt-and-shape inspection using the Cinatra LLM bridge
