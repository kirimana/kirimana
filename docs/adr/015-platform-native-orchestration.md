# ADR 015 — Platform-native orchestration compilation

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** originally ADR 0015 + 0052 (archived)

## Context

Orchestration intent — schedule, trigger, dependencies, retry, cadence — lives in the contract itself as `dca.orchestration.*` customProperties (see ADR 006). Earlier ADRs left open the question of where the scheduler runs. A one-shot CLI plus user-supplied cron works for DuckDB-on-laptop and for demos. It does not work for production Databricks or Fabric, where real orchestration has to integrate with platform-native job engines, billing meters, SLA dashboards, and IAM.

Three architectures were considered. **Kirimana-as-compiler with platform-native runtimes executing** (the chosen path): Kirimana takes the contract graph plus orchestration metadata plus project target, and emits platform-native pipeline definitions (Databricks Workflows bundles, Fabric Data Pipelines, Airflow DAGs, Step Functions ASL). **Kirimana-ships-its-own-runtime** (rejected): a central orchestrator service that listens for triggers and kicks off `apply` — turns Kirimana into SaaS-like infrastructure, violating the self-hostable premise. **Hybrid via Airflow-or-Dagster framework adapter** (rejected): couples scheduling to transformation and forces users to run Airflow even when Databricks Workflows is already there.

Three further gaps surfaced once basic single-task compilation was working: operators needed multi-task DAGs (fan-out, per-task retry, conditional execution); scheduling needed validation against the lineage graph (so a 09:00 step does not silently follow a step that cannot finish before 10:30); and run-history was not collected anywhere, so drift detection and optimisation suggestions had no baseline.

This ADR merges both decisions: orchestration is a third adapter category, and the compiled artefact is a multi-task DAG with scheduling, validation, and run-history.

## Decision

**`OrchestrationAdapter` is the third adapter category. Kirimana compiles contracts plus orchestration metadata into platform-native pipeline definitions; the platform's own scheduler executes them. The compiled artefact is a multi-task DAG with explicit scheduling, validation, and persisted run history.**

### Three adapter categories

| Category | Answers | Examples | Output |
|---|---|---|---|
| `PlatformAdapter` | Where does the data live? | localduckdb, databricks, fabric | Schema DDL, MERGE SQL, introspection |
| `FrameworkAdapter` | How is it produced / tested? | dbt, sqlmesh, great_expectations | Transformation project files |
| `OrchestrationAdapter` | When, triggered by what, with what failure semantics? | databricks_workflows, fabric_pipelines, airflow, step_functions | Pipeline definition (JSON / YAML / Python) |

### DAG model

The adapter surface carries a multi-task shape that maps 1:1 to the Databricks Jobs API task list and translates to Argo Workflow CRDs or Airflow DAGs:

```python
class WorkflowTask(BaseModel):
    task_key: str
    task_type: Literal["notebook", "python_wheel", "spark_python", "sql", "run_workflow"]
    payload: dict[str, Any]
    depends_on: list[str] = []
    run_if: Literal["all_success", "all_done", "none_failed",
                    "at_least_one_success", "all_failed"] = "all_success"
    retry_attempts: int = 0
    retry_min_interval_seconds: int = 60
    timeout_seconds: int = 3600
    cluster_ref: str | None = None
    tags: list[str] = []
```

### Compilation from contracts

A compiler turns contracts plus selection plus layer cut plus flow overrides into a runnable DAG. Nodes are contracts. Edges come from column-level lineage plus medallion edges. Selectors use dbt-style syntax (`tag:critical`, `+sales_orders`, `path:bronze/crm/`, `layer:silver`). Layer cuts (`target_layer="silver"`) remove nodes after silver at compile time. Flow overrides in `flows/<name>.yml` set per-task `run_if`, retry, cluster, tags, and add cross-contract dependencies the lineage graph does not capture.

### Run modes

| Mode | Semantics |
|---|---|
| `scheduled` | Each task waits for both its parents and its own contract's `start_time` |
| `cascade` *(default)* | Only the first task uses its schedule; downstream tasks fire from `depends_on` |
| `manual` | No schedule; CLI / UI / API trigger only |

### Compile-time validation

Three severity levels (ERROR blocks deploy, WARN shows in CLI and UI, INFO is the assistant narrating decisions): impossible arithmetic (child starts before parent can finish), SLA shorter than the critical path, cycles, broken `depends_on` references, lineage gaps, cron expressions hitting DST gaps, retry budgets that blow the SLA. Validation runs at `kiri compile`, `kiri lint`, and `kiri apply --plan`.

### Run-history store

Every task run lands in Postgres (the same database the release-tracking layer uses, see ADR 010). The AI Gateway (see ADR 016) consumes the history for drift detection, parallelism suggestions ("tasks A and B always run sequentially but lineage proves them independent — parallelising saves 22 minutes"), and cluster right-sizing. All suggestions land as PR proposals — never auto-applied.

### AI boundary

AI operates at the metadata layer — drafting `dca.orchestration.*` blocks from natural language, narrating cadence reconciliation, generating runbooks. AI does not write pipeline JSON or YAML directly: pipeline syntax is too brittle and platform-specific for probabilistic generation to be safe. Deterministic compilation is cheaper and auditable.

## Consequences

- **Positive:** Zero new runtime for Kirimana — platforms' own schedulers are battle-tested and billing-integrated; platform-native observability inherits transparently; one contract metadata shape, N targets — switch targets by changing one field; deterministic auditable compilation (pipeline diffs are reviewable in PRs); multi-cloud is emergent rather than bolted on.
- **Negative / costs:** N adapters must be written, tested, and maintained (mitigated because pipeline formats are mostly declarative); emission-time is not runtime, so `dca.orchestration.*` changes require a re-emit and deploy to take effect; cross-adapter features are subject to the lowest common denominator (escape hatches live under target-prefixed namespaces, documented as non-portable); testing platform-specific adapters is hard without the platform (mitigated by validation CLIs and conformance fixtures).
- **Neutral:** The `manual` adapter covers local dev and CI with no regression for existing users; conditional execution, data-dependency triggers beyond schedule + event, and SLA-based retry policies are deferred — the namespace reserves shape for them.

## Related

- See [arch-09-integrations](../architecture/arch-09-integrations.md)
- Related ADRs: 006, 010, 011, 012, 016
