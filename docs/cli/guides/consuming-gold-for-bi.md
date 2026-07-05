# Consuming gold for BI

**Audience:** BI specialists and analytics engineers. **Read this** to consume Kirimana's
gold layer the governed way — one semantic definition every BI tool shares, KPIs with
lineage and freshness, data-SLA enforcement, localized twins for local-language reporting,
and a maturity score to know where you stand.

## One semantic definition, every BI tool

Kirimana's gold contracts declare their measures and dimensions (`kiri.gold.measures`,
`kiri.gold.dimension_refs`). That declaration is the *single source of truth* for a metric —
so Power BI, Tableau, Looker, and Cube all compute a KPI the same way instead of each team
re-defining "revenue" in its own tool.

Emit and apply the semantic layer:

```bash
# Emit semantic-layer YAML from gold measures + dims
kiri semantic sync --backend databricks-metric-views -o semantic/models.yml
kiri semantic sync --backend cube -o semantic/cube.yml        # or metricflow

# Apply gold semantic models to Unity Catalog as Metric Views (dry-run by default)
kiri semantic apply --backend databricks-metric-views --target prod
kiri semantic apply --backend databricks-metric-views --target prod --no-dry-run

# Diff live Metric Views against the contracts
kiri semantic verify --backend databricks-metric-views --target prod --fail-on all
```

`kiri semantic sync` supports `cube`, `metricflow`, and `databricks-metric-views`; the first
two write vendor-neutral YAML you commit to the BI repo. `apply` and `verify` currently
target Unity Catalog Metric Views. `apply` is dry-run unless you pass `--no-dry-run` (which
also requires `KIRIMANA_SEMANTIC_VIEWS_LIVE=1` and a unity-catalog target). Run `verify` in
CI with `--fail-on missing|stale|all` to catch drift between the deployed metric views and
the governed contracts.

## KPIs with lineage and freshness

A KPI is not just a number — it has an owner, upstream lineage, a freshness state, an SLA,
and a cost. Look them up directly:

```bash
kiri kpi list                              # every KPI discoverable in the project
kiri kpi describe monthly_revenue          # lineage, owner, freshness, SLA, cost
kiri kpi describe monthly_revenue --format json
kiri kpi describe monthly_revenue --no-freshness   # faster; skips the manifest read
```

`kiri kpi describe` is what you show an executive who asks "can I trust this number and how
fresh is it?" — it answers with the KPI's lineage back to source and its current freshness.

## Data-SLA and freshness enforcement

Freshness and quality guarantees are declared as SLAs on contracts (`kiri.schedule.sla_deadline`
and the ODCS `sla[]` block). `kiri sla check` evaluates them across the project and routes
breaches:

```bash
kiri sla check --target prod                       # check + route breaches
kiri sla check --target prod --contract dim_customer
kiri sla check --target prod --dry-run             # print what would happen, dispatch nothing
```

Run it on a schedule so a stale gold table raises a breach *before* a dashboard shows a BI
consumer yesterday's numbers. `--dry-run` is the safe way to preview routing.

## Localized twins for BI tools

When reports need a local language, consume the **localized serving twins** — read-only
views over the canonical gold table with locale-specific table and column names (e.g. a
Swedish `serving_sv` schema alongside `serving`). Point a local-language report at the twin
schema; the English table stays the single source of truth. Twins are opt-in per table — see
[gold star schema & localized twins](./gold-star-schema-and-localized-twins.md) for how they
are declared.

## Know your maturity

Before investing in a BI accelerator, score where the project stands against the Gartner BI
maturity model:

```bash
kiri maturity assess                       # text scorecard + advisory
kiri maturity assess --format markdown
kiri maturity assess --require-level 3     # exit non-zero in CI if below level 3
```

`--require-level` lets a platform commit to a maturity floor and gate merges against it, so a
regression in governance coverage or semantic activation is caught at PR time.

## A typical BI onboarding

1. `kiri maturity assess` — baseline.
2. `kiri semantic sync` + `kiri semantic apply` — publish governed metrics to the BI tools.
3. `kiri kpi list` / `kiri kpi describe` — confirm each headline KPI has lineage + freshness.
4. `kiri sla check` on a schedule — enforce freshness before consumers see stale data.
5. Point local-language reports at the localized twins.

## Related reference

- [Gold & analytics](../gold-analytics.md) — the generated `kiri semantic` / `kiri kpi` / `kiri sla` / `kiri maturity` reference
- [Gold star schema & localized twins](./gold-star-schema-and-localized-twins.md)
- [Column & goal lineage](./lineage-column-and-goal.md)
