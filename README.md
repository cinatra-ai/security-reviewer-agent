# Security Reviewer Agent

A security-review helper that sits beside the agent-authoring chat experience. It reads a draft agent definition and flags the fuzzy risks a deterministic linter cannot see — prompt-injection openings, over-broad tool permissions, missing authorization checks, suspicious external hosts, and credentials or personal data sneaking into prompts.

## Capabilities

- Spot prompt-injection openings in agent prompts
- Flag over-broad tool permissions and missing scope checks
- Catch credentials or personal data leaking into system or user fields
- Surface suspicious external host references
- Recommend approval gates on risky write actions
