# Technology Stack

**Analysis Date:** 2026-06-09

## Languages

**Primary:**
- TypeScript — source code in `src/` (compiled to `dist/`); `tsconfig.json` targets ES2023 with ESNext modules
- JavaScript (ESM) — zero-dependency CI gate script `extension-kind-gate.mjs` (Node builtins only)

**Secondary:**
- JSON — agent specification format; `cinatra/oas.json` and `skills/security-review-methodology/SKILL.md` are the primary deliverables

## Runtime

**Environment:**
- Node.js 24 (specified in `.github/workflows/ci.yml` via `actions/setup-node@v4`)

**Package Manager:**
- pnpm (corepack-managed; `.npmrc` sets `auto-install-peers=false`)
- Lockfile: not present in repo (source mirror — monorepo owns install)

## Frameworks

**Core:**
- Cinatra OAS Flow 26.1.0 — agent definition format; the repo ships a single-flow agent declared in `cinatra/oas.json`
- No runtime web framework — this is a Cinatra platform agent, not a standalone HTTP server

**Testing:**
- Not detected — no test runner config found; the CI gate (`extension-kind-gate.mjs`) serves as the validation layer

**Build/Dev:**
- TypeScript compiler (`tsc`) — config in `tsconfig.json`; outputs to `dist/`; `rootDir: "src"`
- Corepack — manages pnpm version for CI

## Key Dependencies

**Critical:**
- None declared in `dependencies` or `devDependencies` in `package.json` — the repo is a source mirror; the Cinatra monorepo provides all `@cinatra-ai/*` packages as optional peer dependencies

**Infrastructure:**
- `extension-kind-gate.mjs` — self-contained Node-builtins-only CI gate; validates `cinatra/oas.json` parses correctly and scans for retired CRM primitives in LLM-visible prompt strings

## Configuration

**Environment:**
- No `.env` files detected
- Runtime secrets injected via Cinatra platform at execution time (e.g., `CINATRA_BASE_URL` referenced as `{{CINATRA_BASE_URL}}` in `cinatra/oas.json`)

**Build:**
- `tsconfig.json` — strict TypeScript, ESNext module, `bundler` module resolution, JSX react-jsx, emits `declaration` + `declarationMap` + `sourceMap` to `dist/`
- `.npmrc` — `auto-install-peers=false`

## Platform Requirements

**Development:**
- Node.js 24+
- pnpm (via corepack)
- Monorepo workspace context required for full install/typecheck (repo is a source mirror; standalone install intentionally skipped when first-party peers are declared)

**Production:**
- Cinatra platform runtime (agent executed by Cinatra OAS Flow engine 26.1.0)
- OpenAI API access via Cinatra LLM bridge (`preferredProvider: "openai"`, `preferredModel: "gpt-5.5"`)

---

*Stack analysis: 2026-06-09*
