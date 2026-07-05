# Orchestration & scheduling

**Audience:** platform architects and data-platform engineers. **Read this** to schedule and
operate Kirimana pipelines the platform-native way — declaring schedules as contract
metadata, compiling contracts into native jobs, and managing the flows lifecycle.

## Schedules are contract metadata

Kirimana does not run its own scheduler. A schedule is *declared* on a contract under the
`kiri.schedule.*` namespace and *compiled* into the platform's native job system (for
Databricks, a Jobs 2.1 workflow). The metadata is portable; the compiled artefact is
platform-specific.

Common keys:

| Key | Purpose |
|---|---|
| `kiri.schedule.cadence` | cron expression / cadence for the flow |
| `kiri.schedule.timezone` | timezone the cadence is evaluated in |
| `kiri.schedule.start_time` | first eligible run time |
| `kiri.schedule.catchup` | whether to backfill missed intervals |
| `kiri.schedule.dependencies` | upstream flows this one waits on |
| `kiri.schedule.retry` | retry policy |
| `kiri.schedule.sla_deadline` | deadline used by SLA checks |
| `kiri.schedule.expected_duration_minutes` | baseline for anomaly detection |
| `kiri.schedule.trigger` / `event_source` | event-driven (vs cron) triggering |

Because the schedule lives on the contract, the same declaration produces a native job on
any supported runtime.

## Compile contracts to native jobs

The compiler walks the contract graph — column-level and medallion lineage — and emits a
`WorkflowSubmission` whose tasks map 1:1 onto the platform's task DAG. Layer cuts,
dbt-style selectors (`+name`, `name+`, `tag:foo`, `layer:silver`, `domain:sales`), and
medallion-state filters all apply, and `paused` / `deprecated` / `draft` contracts drop out
with a machine-readable reason.

```bash
# Compile one medallion layer of a domain to runnable artefacts
kiri compile layer silver --domain sales --target prod

# Compile a depth-range × breadth-selector slice (closure-aware)
kiri compile range --domain sales --target prod --from bronze --upto gold
kiri compile range --domain sales --target dev --from silver --upto silver \
  --include 'silver.SAT_*' --dv-kind satellite
```

`kiri compile layer` accepts a `--compile-target` override (e.g. `sql`,
`dlt-with-expectations`, `dlt-with-apply-changes`, `view`, `materialized-view`); the
`apply-changes` targets require `--project-adapter`. `kiri compile range` adds
closure-completeness — by default it auto-expands the selected set to include dependencies;
`--no-closure` refuses and reports what is missing instead.

## The flows lifecycle

Once compiled, scheduled flows are managed against the live platform. Declared flows live
under `flows/*.yml` and are reconciled to the platform with `sync`:

```bash
# Reconcile flows/*.yml with the platform (diff first, then apply)
kiri flows sync --target prod --plan-only
kiri flows sync --target prod
kiri flows sync --target prod --prune          # delete platform jobs not in YAML

# List, pause, resume, trigger
kiri flows list --target prod
kiri flows pause  <flow-name> --target prod
kiri flows resume <flow-name> --target prod
kiri flows trigger <flow-name> --target prod   # fire now, ad-hoc
```

`--plan-only` prints the diff without applying; `--dry-run` compiles job specs without
contacting the platform at all — useful in CI to catch a bad schedule before it ships.

## Observe and tune

```bash
# Recent runs from the flow_runs history table
kiri flows runs <flow-name> --window-days 30 --limit 50

# Kiri's recommendations for a flow (right-sizing, retry, schedule tuning)
kiri flows recs <flow-name> --window-days 30
kiri flows recs <flow-name> --json
```

`kiri flows recs` reads the run history and surfaces tuning suggestions — for example a flow
whose observed duration drifts past its `expected_duration_minutes`, or a retry policy that
never helps. Feed accepted recommendations back into the contract's `kiri.schedule.*` block
and re-`sync`.

## A typical loop

1. Declare `kiri.schedule.*` on the relevant contracts.
2. `kiri compile layer` / `kiri compile range` to generate artefacts; verify with
   `kiri flows sync --dry-run`.
3. `kiri flows sync --plan-only` then `kiri flows sync` to reconcile the platform.
4. `kiri flows runs` + `kiri flows recs` to observe and tune; loop.

## Related reference

- [Orchestration & scheduling](../orchestration-scheduling.md) — the generated `kiri flows` / `kiri compile` reference
- [The medallion + contracts model](./medallion-and-contracts-model.md) — the contract graph the compiler walks
- [Column & goal lineage](./lineage-column-and-goal.md) — the lineage that drives task edges
