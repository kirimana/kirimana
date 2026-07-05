# Gold star schema & localized serving twins

**Audience:** platform architects and analytics engineers. **Read this** to model the gold
layer as a Kimball star schema Kirimana can generate and govern, and to serve it in more
than one language through per-table localized view twins.

## Gold is a governed star schema

Kirimana's gold layer is an explicit Kimball star: facts and dimensions with declared grain,
measures, surrogate keys, and dimension references. The star is not inferred — it is
metadata on each gold contract under the `kiri.gold.*` namespace:

| Property | Meaning |
|---|---|
| `kiri.gold.kind` | `dimension` \| `fact` \| `factless_fact` \| `snapshot_fact` \| `accumulating_snapshot` |
| `kiri.gold.grain` | the fact's declared grain |
| `kiri.gold.measures` | additive / semi-additive measures (also feed the semantic layer) |
| `kiri.gold.dimension_refs` | which dimensions the fact joins to |
| `kiri.gold.scd_type` | for dimensions: `type_1` / `type_2` / `type_3` / `type_7` |
| `kiri.gold.surrogate_key_column` / `surrogate_key_strategy` | `hash` / `sequence` / `natural` |

Because grain and measures are declared, the same gold contract drives both the physical
build *and* the semantic layer that BI tools consume — see [consuming gold for BI](./consuming-gold-for-bi.md).

## Drafting gold with Kiri

Kiri drafts one fact or dimension of a reporting goal at a time. You accept, edit, and
commit — the AI proposes, you own:

```bash
# Draft a dimension (SCD type 2 for history) from a reporting goal
kiri suggest gold --goal monthly_sales --dim customer --scd-type type_2 --write

# Draft a fact at transaction grain
kiri suggest gold --goal monthly_sales --fact sales --fact-kind transaction --write
```

Notes:

- `--goal` names a `ReportingGoal` under `models/gold/`; one of `--fact` / `--dim` is
  required, drawn from that goal's declared `spec.facts` / `spec.dimensions`.
- `--scd-type` defaults to `type_1` for safety — bump to `type_2` when you need history.
- `--surrogate-key-strategy` defaults to the project's silver technique (Data Vault → `hash`,
  Kimball → `sequence`, flat → `hash`).
- `--write` saves under `models/gold/` as `<goal>__<name>.sql`; `--overwrite` replaces an
  existing file. Without a configured AI gateway, `--allow-stub` emits an audited template.

## Verbatim & bilingual naming

Two naming controls keep generated gold aligned with real-world conventions:

- **Verbatim upstream refs.** On bronze contracts, `kiri.naming.bronze_ref_verbatim` makes
  downstream `ref()` use the bare `<table>` name instead of the `<source>__<table>` form,
  so generated SQL references the object names the source system actually uses.
- **Bilingual annotations.** Columns and tables can carry a localized name and comment
  (`sv_name` / localized `COMMENT`) alongside the canonical English name, so the catalog and
  the warehouse both show the business-facing label without renaming the physical object.

## Localized serving twins

A single English gold layer is often not enough — a Nordic analyst needs Swedish column and
table names, a Power BI report may need the local vocabulary. Kirimana solves this with
**localized view twins**: read-only views over the canonical gold table, renamed and
re-commented per locale. The English table stays the single source of truth; the twin is a
serving convenience.

Twins are configured layer-wide on `01-config/<domain>/gold.yml` under `kiri.localization.*`:

| Key | Purpose |
|---|---|
| `kiri.localization.locales` | which locales to emit (e.g. `[sv]`) |
| `kiri.localization.default_locale` | the canonical locale |
| `kiri.localization.schema_pattern` | how the twin schema is named — e.g. `serving` → `serving_sv` |
| `kiri.localization.names` | the localized table/column names |
| `kiri.localization.descriptions` | localized comments |
| `kiri.localization.on_missing` | `fail` (require full coverage) or `fallback` |

### Per-table opt-in

Localization is **opt-in per table**, because real EDWs are mixed — a few dimensions have
localized twins, many are English-only. The rule keys on a single signal:

> A gold table participates in localization — and gets a twin — **only if it declares a
> table-level `kiri.localization.names`**. A table without it is *canonical-only*: no twin
> is generated, and it is exempt from locale coverage even under `on_missing: fail`.

This avoids two failure modes the older all-or-nothing behaviour caused: failing validation
when you add an un-localized dimension, or fabricating a *phantom* twin that has no
counterpart in the source to reconcile against. Declare `kiri.localization.names` on the
tables you want twinned; leave it off the rest.

The twin schema name comes from `schema_pattern` — a `serving` layer with locale `sv`
produces a `serving_sv` schema whose views mirror `serving`, renamed per the `names` map.

## Related reference

- [Gold & analytics](../gold-analytics.md) — the generated `kiri suggest gold` / gold command reference
- [Consuming gold for BI](./consuming-gold-for-bi.md) — semantic layer, KPIs, SLAs over this star
- [Naming, catalog & schema registry](./naming-catalog-and-schema-registry.md) — the schema/catalog registry twins route into
- [The medallion + contracts model](./medallion-and-contracts-model.md)
