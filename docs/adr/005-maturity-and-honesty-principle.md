# ADR 005 — Maturity and honesty principle

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** originally ADR 0044 (archived)

## Context

A platform that ships many features at different stages of development has two ways to communicate maturity: pretend everything is production-ready and let users find the seams the hard way, or label honestly and let users self-select. The first approach wins short-term demos and loses long-term trust. The second approach is the only one compatible with serving regulated buyers, with a credible standards-authorship posture (see ADR 008), and with the brand that lets the team recruit engineers who care about correctness.

Kirimana also serves two segments with very different governance maturity: large regulated organisations with formal centres of excellence, per-domain stewards, and approval matrices; and smaller teams where one engineer is also the steward, the approver, and the operator. A single forced governance posture for both is wrong in both directions — heavy two-approver gates make the product unusable for small teams, and light single-approver flows make it non-compliant for the regulated tier.

The honesty principle generalises beyond governance modes. Every adapter surface (see ADR 011), every framework integration, every UI surface, and every adapter implementation lives somewhere on a maturity ladder — and that position needs to be visible to users, not hidden in commit history.

## Decision

**Every public-facing capability of Kirimana ships with an explicit maturity tier, visible on the public maturity page at `kirimana.io/maturity`.** Four tiers:

- **Production-ready** — has shipped, has at least two production-grade adapter implementations or equivalent test coverage, passes the conformance suite (see ADR 012), and carries a documented support commitment.
- **Beta** — feature-complete, partial conformance evidence, breaking changes possible at minor-version boundaries with deprecation notes. Safe for non-critical workloads.
- **Experimental** — implemented but not stable; behaviour or shape may change without deprecation. Documented as opt-in.
- **Proposed** — an accepted design (an ADR exists) but no shipping implementation. May change before it ships.

The honesty principle has a second axis: **governance mode**. Projects pick `light` (single-approver) or `enterprise` (two-approver on the destructive action set: catalog push, audit redaction, contract retirement, RBAC-policy change, classification raise, cross-domain consumption). A `dca_core.approval_policy` resolver is the single source of truth for "how many approvers does this action need" — every gating surface (CI workflows, BFF endpoints, runtime checks) consults the resolver rather than hardcoding a count. Routine authoring stays single-approver in either mode, so the centre of excellence does not become a bottleneck on daily work.

Domain-level overrides (a finance domain that demands three reviewers for classification raises) take precedence over the project-wide mode. A lint rule rejects newly added code that hardcodes approval counts.

## Consequences

- **Positive:** The maturity page is a public commitment that disciplines internal planning — promising "production-ready" in marketing means earning the conformance evidence to back it. The two-tier governance switch (light versus enterprise) lets the same product serve a five-person team and a regulated bank without per-feature configuration sprawl.
- **Negative / costs:** Some commercial features will sit at "beta" or "experimental" longer than sales would prefer; that is the price of the brand. Every destructive surface must call the approval resolver rather than hardcoding a count; the audit at landing time required refactoring one site (audit redaction) and the lint rule guards future code.
- **Neutral:** Domain stewards in light-mode projects have RBAC roles but no blocking approval authority; documented in the operator handbook as the deliberate trade-off it is.

## Related

- See [arch-01-overview](../architecture/arch-01-overview.md)
- Related ADRs: 004, 011, 012
