# ADR 008 — `ai_policy` extension proposed to the Bitol working group

- **Status:** Accepted (proposal in flight; spec adoption pending)
- **Date:** 2026-05-13
- **Re-curated from:** originally ADR 0059 (archived)

## Context

Kirimana's `dca.ai_policy.*` customProperties (`training_data`, `pii_exposure`, `egress_rules`) are production code paths today — every LLM call routes through the AI Gateway (see ADR 016), audit-logged, refused fail-closed when the policy says no.

ODCS v3.1.0 (governed by Bitol since 2023-11-30) has no AI-related fields. The spec defines eleven sections (Fundamentals, Schema, References, Data Quality, Support, Pricing, Team, Roles, SLA, Infrastructures, Custom and Other Properties); none address AI policy, model usage, training-data restrictions, PII-exposure controls, or egress rules. No documented namespace under Custom and Other Properties covers AI metadata either. Bitol is a Linux Foundation AI & Data project — AI extensions are within its mission, not orthogonal to it. The slot is open.

The gap persists because the category is recent (AI-governance-as-data-contract-concern emerged through 2023-2024 with the EU AI Act and widespread foundation-model use), because commercial vendors do not open-source their AI-policy differentiators, and because most open-source contract projects are too narrow to have AI-policy concepts in their domain. Kirimana hits the rare four-way intersection: open-source core (see ADR 001), AI policy as a first-class contract concept, community-friendly posture, and the capacity to do spec-body work.

## Decision

**Propose an `aiPolicy` extension to ODCS through the Bitol Working Group, with Kirimana as the reference implementation.**

The proposal standardises three contract-level fields, derived directly from the production code paths in the Kirimana AI Gateway:

| Field | Type | Purpose |
|---|---|---|
| `aiPolicy.trainingData` | enum (`forbidden` / `consented` / `synthetic-only` / `unrestricted`) | Whether columns from this contract may be used as training data |
| `aiPolicy.piiExposure` | enum (`forbidden` / `redacted` / `aggregated` / `permitted`) | What PII handling is allowed when columns are referenced in an LLM prompt |
| `aiPolicy.egressRules` | object (`models`, `regions`, `tenancy`) | Which models, regions, and tenancies may receive data from this contract |

These mirror production semantics already exercised by the Kirimana AI Gateway, so the proposal arrives with a working reference implementation rather than a paper design.

Phased plan:

| Phase | Window | What |
|---|---|---|
| Pre-flight | Q1 2026 | Stabilise `dca.ai_policy.*` semantics; document the namespace as spec-ready |
| Bitol RFC | Q2 2026 | Open RFC against `bitol-io/open-data-contract-standard`; engage the working group |
| Draft adoption | Q3 2026 | Land as `aiPolicy:` in the ODCS v3.2 draft |
| Forward-compat alias | Q3–Q4 2026 | Kirimana reads both the legacy namespace and the new spec name; deprecation warning on the legacy form |
| Reference implementation | Q4 2026 onwards | Public posture: "Kirimana is the reference implementation of ODCS `aiPolicy`" |

The work is conducted in public from day one. RFC drafts, working-group minutes, and the prototype branch are linked from `docs/standards.md`.

## Consequences

- **Positive:** Standards-authorship credibility with LF AI & Data; AI policy becomes a portable user contract rather than a vendor namespace; Compliance Pack capabilities tie cleanly to the spec extension; long-horizon brand permanence ("Google for AI policy data contracts, find Kirimana in spec authorship"); recruiting leverage; investor narrative shifts from "we sell a tool" to "we shape the metadata category".
- **Negative / costs:** Standards-body work has no 90-day revenue payoff — this is a two-to-five-year compounding bet. If runway is short or the team is small enough that 2-4 months of standards work would derail product, this is the wrong investment. Anchored to ADR 001's Apache-2.0-forever commitment, the bet is well-positioned; without that commitment, it would be premature.
- **Neutral:** Final field naming (`aiPolicy` versus `ai_policy` versus `governance.ai`) follows Bitol's existing camelCase convention. Producer-side scope first; AI consumption (models that synthesise downstream) is a follow-up minor extension if the initial proposal lands.

## Related

- See [arch-02-contracts](../architecture/arch-02-contracts.md), [arch-04-ai-gateway](../architecture/arch-04-ai-gateway.md)
- Related ADRs: 001, 006, 016
