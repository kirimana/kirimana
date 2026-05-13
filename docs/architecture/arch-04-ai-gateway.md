# arch-04: The AI Gateway

Kirimana uses language models. It does not let them touch governed assets directly. Every AI call passes through a single internal service — the AI Gateway — which classifies the request, decides whether the call is permitted, executes it, and writes an audit record. This document describes the trust ladder that drives those decisions, the MCP server that exposes the same controls to external AI assistants, and the workflow primitives that let users author repeatable AI tasks.

## The trust ladder

Every AI operation in Kirimana sits on a four-level ladder where Level 3 is structurally absent:

- **Level 0 — Read-only assistance.** Explaining a contract, summarising a lineage chain, describing a plan diff. Output shown to the user; no persistence. The model never sees data, only metadata.
- **Level 1 — Drafting.** Proposing a contract change, drafting a rule, scaffolding an ingest configuration. The model produces a typed proposal that lands as a PR or pending artefact for human review; never auto-applied.
- **Level 2 — Operator-augmented.** Kiri proposes, the user confirms. The action executes only after an explicit user confirmation step; the gateway records both the proposal and the confirmation.
- **Level 3 — Structurally absent.** No autonomous changes. There is no `TrustLevel` constant for Level 3 and no configuration that enables it. The boundary between AI output and system change is always a human action.

The classification is done at the gateway boundary, not by the calling code. A skill or MCP tool cannot self-declare its level; the gateway inspects the requested operation and assigns it. The `TrustLevel` enum in code has values 0, 1, 2 only.

## Classification-gated LLM calls

The gateway is implemented as a typed Python service (`packages/ai_gateway/`) sitting in front of every model provider. It enforces:

- **Default-deny opt-in.** No AI provider is reachable without explicit project-level configuration naming the provider, the credential reference, and the permitted operations.
- **Typed returns.** Every gateway call returns a structured object (a proposed contract, a draft rule, a summary) rather than free-form text. Callers do not parse model output; the gateway parses and validates against a Pydantic model and rejects malformed responses.
- **PII redaction at egress.** Contracts are scanned for fields tagged as PII before they leave the gateway boundary; the AI provider never sees those values. The `aiPolicy` extension proposed in arch-10 makes this declarative.
- **Bounded retries and degraded mode.** If the provider is unreachable, the gateway returns a typed error; calling code must handle the degraded path explicitly (arch-08).

Provider credentials follow the vault-rotation pattern: 90-day default rotation cadence, versioned vault paths, drain window for in-flight calls. The gateway never holds long-lived keys in memory.

## MCP as the external AI surface

Where the AI Gateway is the internal control point, the **Model Context Protocol (MCP) server** is the external one. Any AI assistant that speaks MCP — Claude Code, Cursor, Continue, Zed, Claude Desktop — connects to a Kirimana project as an MCP client and uses a fixed set of tools: `kirimana_check_policy`, `kirimana_validate_contract`, `kirimana_propose_change`, `kirimana_read_catalog`, and so on.

The MCP server enforces the same trust ladder. Tools that would mutate governed assets either propose a change for confirmation (Level 1 or Level 2) or return a refusal where the operation would constitute an autonomous change (Level 3, structurally absent). Every MCP call is audited (arch-06) with the same event shape as internal calls, so a contract change initiated from an external assistant is indistinguishable in audit from one initiated from the CLI.

This is deliberate: MCP is not a back door. It is the same gateway, fronted differently.

## AI-driven ingest onboarding

The first AI-assisted user flow shipped in Kirimana is `kiri ingest new-source --interactive`. The user names a source system; the gateway runs a versioned per-backend prompt that drafts a source contract, suggests connection metadata, and proposes an initial schema. The output is a contract PR, not a live source — Level 1 drafting with explicit human review. The prompt versions are tracked in the repo so the same input produces the same output across releases.

## Claude Code skills as workflow primitives

Repeatable AI tasks are authored as **skills** — Markdown files under a `skills/` directory at one of three scopes (reference, project, organization). A skill bottoms out at the CLI or MCP tool layer; it does not bypass the gateway. Skills are versioned in the same repository as the contracts they operate on, which means workflow logic, contract content, and audit evidence sit in the same commit history.

## Related ADRs

008, 016, 017, 018, 019, 024
