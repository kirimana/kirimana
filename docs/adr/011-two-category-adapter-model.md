# ADR 011 — Two-category adapter model: Platform and Framework

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** originally ADR 0003 (archived)

## Context

The Kirimana core is platform-agnostic. Platform and framework specifics live in adapters. The model has to define: what the canonical model is, what categories of adapters exist, the interface each implements, how adapters are discovered, and how capabilities are declared so the planner can fail cleanly when a contract uses features the target does not support.

A single adapter category would force platform and transformation-framework adapters to share methods that make sense for neither — `apply_ddl` is meaningless for dbt, `emit` is awkward for DuckDB. Many categories would be premature; catalog and BI integrations are not foundation-relevant, and adding categories speculatively constrains designs that have not yet been validated.

## Decision

**Two adapter categories**, each with its own abstract base and capability manifest.

### PlatformAdapter — where does the data live?

Translates the canonical model into operations against a data platform.

Core operations: `read_schema(reference)`, `apply_ddl(plan)`, `run_quality_check(rule, target)`, `read_sample(reference, n)`.

Target platforms include DuckDB (first, see ADR 013), Databricks (flagship, see ADR 004), Microsoft Fabric, Snowflake, PostgreSQL, BigQuery.

### FrameworkAdapter — how is the data produced and tested?

Translates the canonical model into artefacts for a transformation, orchestration, or quality framework.

Core operations: `emit(canonical, target_path)`, `describe_outputs()`.

Framework adapters are write-mostly in the foundation phase (generate, do not parse back). Targets include dbt-core (first), SQLMesh, Dagster, Airflow, Great Expectations.

### Shared: capability manifests

Every adapter declares a capability manifest. The planner checks each contract against the target adapter's declared capabilities and produces a clear error at plan time for unsupported feature usage:

```yaml
adapter:
  name: duckdb
  category: platform
  version: 0.1.0
  odcs_versions_supported: ["3.0"]
  capabilities:
    read_schema: true
    apply_ddl: true
    run_quality_check: true
    read_sample: true
  unsupported_odcs_features:
    - sla.targets.freshness
```

### Discovery and conformance

Adapters load via Python entry points under `dca.platform_adapters` and `dca.framework_adapters`. The core never imports a concrete adapter directly. Third parties ship adapters without touching the core repository. Every adapter passes a conformance test suite (see ADR 012) before being listed as supported.

### Canonical model boundary

ODCS YAML is the on-disk format (see ADR 006); the immutable `CanonicalContract` Pydantic v2 model is the in-memory shape. Every adapter, generator, AI provider, and UI consumes the canonical model and never parses YAML itself.

Orchestration grows into a third category in its own ADR (see ADR 015); the two-category model is the foundation, and additional categories are added explicitly when concrete demand warrants them.

## Consequences

- **Positive:** Core has zero hard dependencies on any platform or framework — testable in isolation; adding a new platform or framework is "write a plugin", not "fork the project"; capability declarations produce clear errors at plan time rather than cryptic runtime failures; the two-category split reflects actual conceptual differences and keeps each interface tight; third-party vendor adoption is realistic rather than aspirational.
- **Negative / costs:** The plugin architecture adds indirection — debugging a misbehaving adapter requires understanding the plugin-loading path; capability manifests are easy to write incorrectly (the conformance suite is the guard); the core's plugin contract is now a versioned public API, so breaking changes require deprecation cycles (see ADR 012).
- **Neutral:** Python entry points commit the plugin model to Python conventions; a subprocess or RPC model for non-Python adapters is a future ADR if it becomes necessary.

## Related

- See [arch-03-adapters](../architecture/arch-03-adapters.md)
- Related ADRs: 004, 006, 009, 012, 013, 015
