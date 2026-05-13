# ADR 016 — AI Gateway with trust ladder

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** originally ADR 0005 (archived)

## Context

Kirimana is AI-first but not AI-dependent. That is a vector, not a destination. Concretely: where in the architecture does AI sit, which models are supported, which workflows are AI-augmented, what is the fallback when AI is unavailable or forbidden by the deployment environment, how is AI behaviour audited, and how is AI prevented from taking actions that require human approval.

AI in data tooling has a credibility problem. Most "AI-powered" tools either bolt on a chat interface as a feature checkbox, or make AI so central that the deterministic backbone disappears. Kirimana does neither.

## Decision

**AI is a first-class service layer behind a stable abstract base class. Every AI invocation passes through the AI Gateway, returns typed structures (never free-form text), and sits on an explicit four-level trust ladder where Level 3 is structurally absent.**

### Service layer and providers

The `AIService` ABC in the AI Gateway package is the single interface every AI-consuming component depends on. Concrete providers (Anthropic, local via Ollama, future OpenAI, Azure) are plugins implementing the ABC. No component depends on a provider directly. Foundation providers are `anthropic` (audited through the gateway with prompt caching, classification enforcement, audit log) and `local` (Ollama or compatible runtime — air-gapped deployments are first-class, not an afterthought).

### Typed returns

The ABC's methods return typed result classes — `ContractDraft`, `RuleDraft`, `Explanation`, `DriftFinding`. Free-form text never escapes the AI boundary. Structural validation runs before any result is shown.

### Disclosure and default-deny

Every provider declares `ProviderCapabilities` with `data_egress` (where does the data go, which region, which service). Privacy-sensitive workspaces can require `sends_to_external_service: false` — only the `local` provider satisfies that constraint. New workspaces default to AI-disabled. Enabling AI requires listing allowed providers in `.kirimana/ai_policy.yml`. This is a deliberate trust signal that distinguishes Kirimana from "AI by default" tooling.

### The trust ladder

| Level | Operation type | Examples | Default behaviour |
|---|---|---|---|
| 0 | Read-only assistance | `explain` of plan diff, contract description | Output shown to user; no persistence |
| 1 | Suggestion for review | `suggest_contract_from_schema`, `translate_rule` | Output typed; user explicitly accepts before any write |
| 2 | Structured detection | `detect_drift` | Findings produced; user decides response |
| 3 | **Autonomous mutation** | *(none — forbidden by design)* | AI does not write spec, code, or platform state, ever |

Level 3 is structurally absent. Even high-confidence AI does not auto-apply, auto-edit, or auto-resolve. The boundary between AI output and system change is always a human action. The `TrustLevel` enum in code has values 0, 1, 2 — Level 3 has no constant.

### Determinism boundary

AI is **forbidden** in the generation pipeline (`plan`, `apply`, `verify` — see ADR 009), in validation-rule execution, in DDL planning, in diff computation, and in conformance tests. AI is **permitted** in contract authoring, documentation drafts, drift findings, and plan-diff explanation. The boundary is enforced architecturally — the generation packages have no import path to `AIService`, and a CI check fails the build on any violation.

### Auditability and reproducibility

Every AI invocation produces a structured log entry under `.kirimana/ai_log/<date>.jsonl`: timestamp, operation, provider, model id, pinned version, prompt hash, response hash, tokens, cost, human decision, resulting artefact path. The log is queryable via `kiri ai history`, `kiri ai cost`, `kiri ai audit`. Prompts and responses are stored content-addressed. Kirimana promises **invocation reproducibility** (same prompt and model id pinned) but not **output reproducibility** — model outputs drift as providers update models, and the platform is honest about that boundary rather than over-promising.

### Caching, budgets, and policy

Identical prompts within a session return cached results. Persistent cache is opt-in per workspace and invalidates on model-id change. Per-workspace budgets (`monthly_usd_max`, `per_invocation_usd_warn`) gate spend. Per-contract `ai_policy` (the proposed ODCS extension, see ADR 008) is enforced fail-closed on every call.

## Consequences

- **Positive:** AI-first vision realised without sacrificing determinism — Kirimana functions identically without AI, just with less acceleration; provider-agnostic with no vendor lock-in; air-gapped deployments are first-class; trust ladder makes AI's role explicit (no surprise autonomy); auditable by construction; deterministic generation is architecturally protected.
- **Negative / costs:** Provider parity is not guaranteed — a local model may produce lower-quality drafts than a hosted one (surfaced honestly via capability declarations); typed-structure output requires careful prompt design; the audit log adds storage overhead (mitigated by content-addressed storage and configurable retention); local-model UX needs more setup than a cloud API (accepted cost for the air-gap guarantee); "AI opt-in per workspace" feels like friction to users who expect AI on by default.
- **Neutral:** Reproducibility is bounded by provider behaviour; the trust-ladder framing is borrowed from autonomous-systems literature and needs a short orientation for some users.

## Related

- See [arch-04-ai-gateway](../architecture/arch-04-ai-gateway.md)
- Related ADRs: 006, 008, 009, 015
