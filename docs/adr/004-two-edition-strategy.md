# ADR 004 — Two-edition strategy: Databricks flagship and OSS Edition

- **Status:** Accepted
- **Date:** 2026-05-13

## Context

A platform-agnostic data-contract product has a market-positioning problem. Pitching "we run on every cloud and every warehouse" sounds attractive but is meaningless to the first buyer, who has one cloud and one warehouse. Buyers and analysts want a flagship — the specific environment where the product is unambiguously production-grade and the team's depth shows. At the same time, the platform-agnostic core (see ADR 011) is real engineering, not marketing — its existence is what makes a credible second platform possible later and what underwrites the cross-platform "exit option" customers buy as risk mitigation.

The flagship choice has to match where the team can ship deepest fastest. Databricks is the clearest fit: Unity Catalog provides a mature catalog surface, Workflows give a real platform-native scheduler (see ADR 015), Iceberg and Delta both have first-class runtime support (see ADR 014), and the deployment story (Azure first, AWS on the roadmap) maps cleanly to the cloud-priority order the team can realistically resource.

At the same time, a credible open-source product cannot live only inside Databricks. The OSS Edition exists so that developers can run the full stack locally on DuckDB (see ADR 013), so that smaller teams can adopt Kirimana without a Databricks contract, and so that the community has a way to contribute against an honest platform-agnostic surface rather than a Databricks-only one.

## Decision

**Kirimana ships in two editions, both Apache-2.0, from a single codebase.**

- **Kirimana for Databricks** — the flagship edition. Production-grade against Databricks on Azure today; AWS support is on the roadmap, with the current target window published on `kirimana.io`. Includes deep integration with Unity Catalog, Databricks Workflows, Delta Lake, and Databricks-managed Iceberg. This is the edition the team leads with in enterprise conversations and the one the maturity matrix (see ADR 005) calls production-ready.
- **Kirimana OSS Edition** — the platform-agnostic edition. Runs against DuckDB locally and against any combination of Platform / Framework / Orchestration adapters (see ADR 011) that ship in the open-source repository. Production-ready against the subset of adapter combinations that pass the conformance suite (see ADR 012); other combinations are honestly labelled beta or experimental.

Both editions are built from the same Apache-2.0 source tree. There is no closed code in the flagship edition. The "edition" distinction is a packaging and support boundary, not a license boundary — Databricks customers get a curated configuration, the OSS edition exposes the full adapter surface. Re-licensing to BSL or SSPL to gate the Databricks edition is permanently forbidden under ADR 001.

Roadmap for additional platform editions (Fabric, Snowflake, BigQuery) is open but unscheduled. Each requires a credible adapter implementation (see ADR 011) plus the conformance evidence (see ADR 012) before the team labels it production-grade.

## Consequences

- **Positive:** Clear flagship narrative for analysts and enterprise buyers; honest community surface for smaller teams and contributors; the two editions reinforce each other (OSS Edition keeps the abstraction honest, flagship keeps the engineering depth visible).
- **Negative / costs:** Twice the support surface — adapter combinations that work in OSS Edition may not be on the supported matrix for paying Databricks customers, and the maturity labels need to stay accurate. Pressure to keep the OSS edition feature-parity with the flagship has to be resisted; honest tier labelling is the answer, not feature gating.
- **Neutral:** Cloud rollout sequence (Azure first, AWS on the roadmap) is a resourcing call, not an architectural one — the adapters are cloud-neutral.

## Related

- See [arch-01-overview](../architecture/arch-01-overview.md), [arch-03-adapters](../architecture/arch-03-adapters.md)
- Related ADRs: 001, 005, 011, 013, 014
