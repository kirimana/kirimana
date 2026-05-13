# ADR 027 — Quality engine integration path (Great Expectations + DQX)

- **Status:** Proposed — backlogged
- **Date:** 2026-05-13
- **Re-curated from:** original ADR 0051 (archived)

## Context

Kirimana's quality story today has three disconnected pieces:

- **`expectations.*` customProperties** on every contract describe column-level invariants in metadata, but nothing executes them at runtime.
- **Data Vault silver-quality checks** cover hub/link/sat referential integrity via dialect-rendered SQL. They are structural and don't cover row-level invariants.
- **Audit-chain coverage** measures Kirimana ↔ platform audit join completeness, not data quality.

The gap is **row + column quality at the bronze/silver boundary**, executed natively on the target platform's compute. Two engines are first-class candidates: Great Expectations (mature, multi-platform) and Databricks Labs' DQX (Databricks-native, lighter operational footprint on the Databricks edition).

A single quality surface in the contract, two pluggable engine implementations behind it.

## Decision

A `QualityEngine` Protocol lives in `packages/quality/`. Two implementations:

- **`GreatExpectationsEngine`** — for the OSS Edition and any non-Databricks target. Compiles `expectations.*` into a GE suite, runs it against the target via the platform adapter.
- **`DqxEngine`** — for the Databricks Edition. Compiles `expectations.*` into a DQX YAML ruleset, runs it via the Databricks SQL Warehouse the project's adapter already binds.

Both return a uniform `QualityResult` (per-rule pass/fail, sample violations capped at 100 rows). Reaction is configurable per contract: `drop | mark | quarantine | fail`, default `quarantine`.

A separate concern — drafting contract-aware rules from sample profiles — runs through the AI Gateway as a PR proposal, never an auto-apply.

## Consequences

- **Positive:** Contract authors describe quality once; the engine plugs in based on edition. Compliance reports aggregate results uniformly across both engines.
- **Negative / costs:** Two engines means two compatibility surfaces to maintain. DQX is younger; some GE rules may not have direct DQX equivalents and require expression-rule fallbacks.
- **Neutral:** Future engines (Soda, Monte Carlo) plug in behind the same Protocol without contract changes.

## Related

- See [arch-07-compliance.md](../architecture/arch-07-compliance.md)
- Related ADRs: 006, 016 (AI Gateway gates rule-drafting), 025 (evidence taxonomy)
