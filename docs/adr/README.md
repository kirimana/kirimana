# Architectural Decision Records

This directory holds the curated set of architectural decisions for Kirimana. Each ADR captures one significant choice — context, decision, and consequences — with a stable number and date.

## How to read these

Start with [`docs/architecture/`](../architecture/) for a current view of what Kirimana *is*. Come back here when you want to know *how we decided*.

## Index

### Foundation & governance
- [001 — Apache-2.0 license, forever](001-apache-2-0-license-forever.md)
- [002 — Governance and stewardship](002-governance-and-stewardship.md)
- [003 — DCO-only contribution model](003-dco-only-contributions.md)
- [004 — Two-edition strategy](004-two-edition-strategy.md)
- [005 — Maturity and honesty principle](005-maturity-and-honesty-principle.md)

### Contract model
- [006 — ODCS v3 as canonical metadata model](006-odcs-v3-canonical-metadata.md)
- [007 — Contract lifecycle and state machine](007-contract-lifecycle-state-machine.md)
- [008 — ai_policy extension (proposed to Bitol WG)](008-ai-policy-extension-proposal.md)
- [009 — Plan / apply / verify lifecycle](009-plan-apply-verify-lifecycle.md)
- [010 — Forward-only environment promotion](010-forward-only-promotion.md)

### Adapter architecture
- [011 — Two-category adapter model](011-two-category-adapter-model.md)
- [012 — Adapter conformance versioning](012-adapter-conformance-versioning.md)
- [013 — DuckDB-first local development](013-duckdb-first-local-development.md)
- [014 — Iceberg as first-class storage primitive](014-iceberg-first-class-storage.md)
- [015 — Platform-native orchestration compilation](015-platform-native-orchestration.md)

### AI integration
- [016 — AI Gateway with trust ladder](016-ai-gateway-trust-ladder.md)
- [017 — MCP as external AI surface](017-mcp-as-external-ai-surface.md)
- [018 — Claude Code skills as workflow primitive](018-claude-code-skills-as-workflow-primitive.md)
- [019 — AI-driven ingest onboarding](019-ai-driven-ingest-onboarding.md)

### Catalog and federation
- [020 — Hub-and-spoke governance catalog](020-hub-and-spoke-governance-catalog.md)
- [021 — Federation resolver](021-federation-resolver-for-cross-project-contracts.md)
- [022 — Catalog pull from external catalogs](022-catalog-pull-from-external-catalogs.md)

### Audit, lineage, compliance
- [023 — Column-level lineage](023-column-level-lineage.md)
- [024 — Audit redaction](024-audit-redaction-for-compliance-reads.md)
- [025 — Quality evidence taxonomy](025-quality-evidence-taxonomy.md)
- [026 — Compliance generators as audit-evidence framework](026-compliance-generators-as-audit-evidence.md)

### Quality and resilience
- [027 — Quality engine integration path](027-quality-engine-integration-path.md)
- [028 — Resilience and degraded-mode operation](028-resilience-and-degraded-mode.md)
- [029 — Rate-limit persistent store](029-rate-limit-persistent-store.md)

### Deployment and runtime
- [030 — Cloud-neutral Helm chart](030-cloud-neutral-helm-chart.md)
- [031 — Project / target isolation](031-project-target-isolation.md)
- [032 — Multi-environment CI/CD](032-multi-environment-cicd.md)

## ADR template

New ADRs follow this shape. Copy and number sequentially. PRs proposing new ADRs are welcomed (see [CONTRIBUTING.md](../../CONTRIBUTING.md)).

```markdown
# ADR NNN — Title

- **Status:** Proposed | Accepted | Superseded by ADR NNN
- **Date:** YYYY-MM-DD

## Context
Why this decision is needed; what forces are in play.

## Decision
What was decided. State the decision plainly.

## Consequences
- **Positive:** ...
- **Negative:** ...
- **Neutral:** ...

## Related
- See [arch-XX-name.md](../architecture/arch-XX-name.md)
- Related ADRs: NNN, NNN
```

## Reserved numbers

Numbers below 100 are reserved for the curated public set. Numbers ≥100 may be assigned to ADRs proposed by external contributors. Internal commercial-track decisions are maintained privately and not exposed here.
