# ADR 013 — DuckDB-first local development

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** originally ADR 0006 (archived)

## Context

The first platform adapter Kirimana ships shapes the entire repository. A complex first adapter (Databricks, Fabric) pushes heavy infrastructure scaffolding into day-one work — cloud accounts, credentials, billing concerns, integration tests against managed services that are slow and flaky. A minimal first adapter (DuckDB) keeps focus on cross-package interfaces rather than vendor SDK plumbing.

The choice also determines the contributor experience. A contributor who can clone the repository, run a single setup command, and exercise the full stack on their laptop will engage. A contributor who has to provision a cloud account, configure secrets, and pay for compute before reading the first error message will not.

## Decision

**DuckDB is Kirimana's first platform adapter. The full stack runs locally on DuckDB with zero cloud dependencies.**

### Why DuckDB first

- **Zero-infrastructure local dev.** Contributors clone, run a single make target, and have the full stack running — no cloud accounts, no credentials, no costs. Foundation of a healthy contribution funnel.
- **CI is fast and free.** The conformance suite (see ADR 012) runs in seconds on a standard runner.
- **Forces honest abstractions.** Building Databricks first would leak Databricks-isms into the core. DuckDB's simplicity is a feature — anything the core needs from it is the irreducible minimum, which becomes the abstraction.
- **Real enough to validate.** DuckDB speaks SQL, handles Iceberg and Parquet, has a credible type system. Generated DDL that works against DuckDB is meaningful evidence that the generator is not just templating strings.

### Second adapter is enterprise-grade, not the fifth

DuckDB lacks features Databricks and Fabric have natively: catalog-level governance, fine-grained access control, multi-region semantics, richer types. The mitigation is structural — the *second* platform adapter is an enterprise platform (Databricks for the flagship edition, see ADR 004), not a fifth toy adapter. Building it early forces the adapter abstraction to express features DuckDB lacks before the codebase ossifies around DuckDB-shaped assumptions.

### Control plane / data plane split

Kirimana has two conflicting requirements: be containerised and easy to deploy, and run inside Databricks / Fabric, which do not execute customer containers for pipelines (they run notebooks and jobs on their own compute). The resolution:

- **Control plane** (orchestration, codegen, CLI, UI, AI gateway) runs as Kirimana containers.
- **Data plane** (SQL execution, DDL apply, ingestion) runs on the target platform.
- Adapters are the bridge: the control plane produces plans; adapters execute them against the data plane.

This keeps Kirimana container-friendly without fighting platforms that have their own execution model.

### Tooling baselines

- Python 3.12 or later.
- Pydantic v2 for every data structure.
- `uv` workspaces for dependency management and linking, with a committed lock file.
- `ruff` (lint and format), `mypy` (strict), `pytest` (tests) — single-tool choices to keep contributor onboarding tight.
- Every source file carries an Apache 2.0 SPDX identifier (CI-enforced).
- Every long-running service ships with a Dockerfile.

## Consequences

- **Positive:** Single clone, single setup command, full local dev — the strongest possible contributor experience; conformance tests run fast in CI without cloud dependencies; DuckDB-first forces clean abstractions before vendor specifics distort them; layout is conventional enough that experienced Python developers feel at home immediately.
- **Negative / costs:** DuckDB-first risks the "first demo runs on a database we do not run in production" perception — countered by the second-adapter discipline; `uv` is newer than Poetry / Hatch, so a migration cost is real if `uv` runs into structural problems (judged mature enough to bet on); monorepo rebuilds on every change (mitigated with per-package CI matrices and incremental test selection).
- **Neutral:** Service packages ship minimal versions in the foundation phase; fuller versions land in later phases.

## Related

- See [arch-03-adapters](../architecture/arch-03-adapters.md)
- Related ADRs: 004, 011, 012
