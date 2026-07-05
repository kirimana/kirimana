# Lineage: column, goal, drift & impact

**Audience:** platform architects and information architects. **Read this** to trace data
two ways — *down* from a business reporting goal to the sources that feed it, and *up* from a
column to everything that depends on it — and to detect when declared lineage drifts from
what the platform actually does.

## Two directions of lineage

Kirimana carries lineage as contract metadata (`kiri.lineage.*`: `upstream`, `downstream`,
`from_columns`, `transformation_kind`, `external_upstream`). That gives it two capabilities
most catalogs lack:

- **Goal-to-data lineage** — a `ReportingGoal` declares the entities it needs; Kirimana
  walks from the goal through gold → silver → bronze → source, so you can answer "does this
  KPI actually have data behind it, and where does it come from?"
- **Column-level lineage** — edges are declared at column grain (`from_columns`), so impact
  analysis is precise: changing one source column tells you exactly which downstream columns,
  contracts, and goals are affected.

## Walk a goal's dependency tree

```bash
# Full dependency tree of one goal — entities, contracts, sources
kiri lineage goal monthly_sales

# Per-goal coverage: how many needed entities actually have contracts
kiri lineage summary
```

`kiri lineage summary` is the fastest way to see which reporting goals are fully backed by
contracts and which have gaps — a goal that needs five entities but has three contracts is
three-fifths served, and the report says so.

## Impact analysis

Before changing a contract, find out who depends on it:

```bash
# List reporting goals that depend on a contract
kiri lineage impact customer
```

For a column-grain trace across the whole pipeline, use the catalog:

```bash
# Trace where a column appears across bronze → silver → gold
kiri catalog show-attribute customer_id
```

This is the reverse of goal lineage: start at a column and see every layer it flows into,
which is what you consult before renaming, retyping, or dropping it.

## Drift detection

Declared lineage and *observed* lineage can diverge — someone hand-edits a model, a
transformation changes, an edge is never recorded. `kiri lineage drift` reconciles the two:

```bash
# Compare declared lineage against an operator-exported observed graph
kiri lineage drift --observed observed_edges.json

# Or read observed lineage live from Unity Catalog's system tables
kiri lineage drift --from-uc --target prod
```

`--observed` takes a JSON file of observed edges (the same wire shape a `LineageEdge`
serialises to — exported from UC `system.access.column_lineage` or OpenLineage).
`--from-uc` reads it live via the configured adapter, no manual export — it reconciles
TABLE-grain lineage (view-materialised layers are not compared). `--max-findings` caps the
report with honest truncation. Run drift in CI to catch a model whose real inputs no longer
match its declared contract.

## Visualise a bounded graph

Lineage graphs get large fast, so `kiri lineage graph` is **bounded by construction** — it
emits a deterministic Mermaid diagram (or machine graph) around a seed URN, never the whole
warehouse:

```bash
kiri lineage graph --urn kiri:asset:silver:customer
kiri lineage graph --urn kiri:asset:silver:customer --direction upstream --depth 3
kiri lineage graph --urn kiri:asset:gold:dim_customer --json     # bounded machine graph
kiri lineage graph --urn kiri:asset:gold:dim_customer --prose    # plain-text explanation
```

`--direction` (`upstream` / `downstream` / `both`), `--depth`, `--max-nodes`, and
`--max-edges` keep the output readable; the node/edge budgets guarantee it never blows past a
size a diagram tool can render. `--prose` emits Kiri's deterministic plain-text walk of the
same graph — useful in a PR description or a review.

## A typical workflow

1. `kiri lineage summary` — which goals are under-served?
2. `kiri lineage goal <goal>` — what feeds a specific KPI?
3. Before a schema change: `kiri lineage impact <contract>` +
   `kiri catalog show-attribute <column>`.
4. In CI: `kiri lineage drift --from-uc` to fence against undeclared lineage changes.
5. For a review: `kiri lineage graph --urn ... --prose`.

## Related reference

- [Lineage & inventory](../lineage-inventory.md) — the generated `kiri lineage` / `kiri catalog` reference
- [Governance & catalog](./governance-and-catalog.md) — findability and the catalog model
- [Orchestration & scheduling](./orchestration-and-scheduling.md) — lineage drives the task DAG
