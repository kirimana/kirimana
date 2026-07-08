# Editing layer manifests — bronze / silver / gold reference

Each layer of a domain is one ODCS v3 `DataContract` document:
`01-config/<domain>/bronze.yml`, `.../silver.yml`, `.../gold.yml`. A manifest lists
that layer's tables and carries the `kiri.*` metadata that drives generation and
governance. This page documents the three levels — **document**, **table**,
**column** — and the properties that live at each.

Validate any change with:

```bash
uv run kiri layer validate --project .          # project-wide, cross-layer
uv run kiri contract validate 01-config/sales/silver.yml --strict
```

The complete `sales` manifests are at `examples/sales/01-config/sales/`.

Field-level semantics for individual `kiri.*` properties are specified in the
contract metadata master:
`https://github.com/kirimana/kirimana/blob/main/docs/contracts/metadata-master.md`.
This page shows **where** each property goes and **how** to set it.

## Document level — the ODCS top matter

Every manifest opens with ODCS-native fields, then `targets`, then
`customProperties` (layer-wide), then `schema` (the tables):

```yaml
apiVersion: v3.0.0
kind: DataContract
id: io.example.sales      # stable domain id (same across the 3 layers)
name: bronze              # the layer: bronze | silver | gold
version: 0.1.0
status: draft             # draft | active | deprecated | retired
domain: sales
owner: data-platform@example.com

targets:
  localduckdb:
    catalog: sales_demo
    governance_schema: sales_governance

customProperties:         # layer-level (apply to every table below)
- property: kiri.layer
  value: bronze
- property: kiri.classification
  value: internal

schema:                   # the tables
- name: sales_order
  ...
```

**`targets`** binds the layer to each environment declared in `kiri.yml`
(reconciled per ADR 0136):

| Field | Notes |
|---|---|
| `catalog` | Target catalog for this layer. Leave blank when `naming.catalog_pattern` renders it in `kiri.yml`. |
| `governance_schema` | Schema for governance/audit objects for this layer. |

**Schema registry (optional, ADR 0134):** a `schemas:` block can declare
logical→physical schema-name mappings when you need explicit names per zone.

## Layer level — `customProperties` on the document

Set the common case once here; tables/columns override as needed.

| Property | Values / shape | Purpose |
|---|---|---|
| `kiri.layer` | `bronze` \| `silver` \| `gold` | **Required.** Identifies the layer. |
| `kiri.classification` | `public` \| `internal` \| `confidential` \| `restricted` | Default data classification for the layer. |
| `kiri.compile_target` | e.g. `sql`, `sql-full-snapshot`, `dlt-with-expectations` | How the layer's models are compiled/materialised. |
| `kiri.dedup.strategy` | e.g. `full_snapshot_scd2` | Bronze dedup strategy. |
| `kiri.schema_evolution.policy` | e.g. `add_to_rescued` | How new/unexpected columns are handled (ADR 0115). |
| `kiri.landing.volume` | path | Bronze landing volume for `read_files()`. |
| `kiri.gdpr.lawful_basis` / `kiri.gdpr.purpose` | string | Layer-wide GDPR defaults (required wherever PII is present). |
| `kiri.localization.*` | see below | Gold: generate localized view twins (ADR 0133). |

### Localization (gold) — and the twin-schema rule

Gold can auto-generate a localized twin schema:

```yaml
customProperties:
- property: kiri.layer
  value: gold
- property: kiri.localization.default_locale
  value: en
- property: kiri.localization.locales
  value: [sv]
- property: kiri.localization.schema_pattern
  value: "{schema}_{locale}"        # serving → serving_sv
- property: kiri.localization.on_missing
  value: fail
```

> **⚠️ Twin-schema collision (ADR 0133 §4.2).** A generated twin schema must be
> **distinct from every canonical schema and every other twin**. If localization
> turns `serving` into `serving_sv`, do **not** also declare a table with
> `kiri.target_schema: serving_sv` — the twin and the canonical would collide and
> `kiri layer validate` fails. Either give the hand-authored table a different
> schema, or model all localized surfaces as explicit entities and drop the
> auto-twin. The [localized-twin task](authoring-tasks.md) shows a clean setup.

## Table level — `customProperties` on each `schema[]` entry

```yaml
schema:
- name: fct_sales_order
  logicalType: table
  customProperties:
  - property: kiri.gold.kind
    value: fact
  - property: kiri.gold.grain
    value: [order_id]
  - property: kiri.source_urn
    value: urn:kirimana:silver:silver_sales_order
  properties:
  - name: order_id
    ...
```

| Property | Where | Purpose |
|---|---|---|
| `kiri.target_schema` | any | Physical schema this table lands in (e.g. `standard`, `curated`, `serving`). |
| `kiri.materialization` | any | `table` \| `view` \| `streaming_table` \| … |
| `kiri.compile_target` | any | Per-table override of the layer's compile target. |
| `kiri.source_urn` | any | URN of the upstream table this one derives from (lineage spine, ADR 0012). |
| `kiri.classification` | any | Table-level classification override. |
| `kiri.gdpr.lawful_basis` / `kiri.gdpr.purpose` | any table with PII | **Required when the table carries PII.** |
| `kiri.semantic.business_keys` | silver | Business key column(s). |
| `kiri.semantic.entity` | silver/gold | Conceptual entity name. |
| `kiri.medallion.silver_zone` | silver | `standard` \| `curated` \| `conformed` (named silver zones, ADR 0137). |
| `kiri.dv.kind` | silver (data_vault) | `hub` \| `link` \| `sat`. |
| `kiri.silver.scd_type` / `kiri.silver.effective_from_column` | silver (scd2) | SCD2 shape. |
| `kiri.transform.sp_source` / `kiri.transform.surrogate_key_recipe` | silver | Ported transform logic / SK recipe. |
| `kiri.transformation.invariants` | any | Post-apply business-rule checks (ADR 0102) — see below. |
| `kiri.gold.kind` | gold | `dimension` \| `fact` \| `factless_fact` \| `snapshot_fact` \| … |
| `kiri.gold.grain` | gold fact | **Required for facts** — columns identifying one row. |
| `kiri.gold.measures` | gold fact | **Required for facts** — list of `{name, column, aggregation, additivity}`. |
| `kiri.gold.dimension_refs` | gold fact | `urn:kirimana:gold:<dim>` references. |
| `kiri.gold.business_keys` / `kiri.gold.surrogate_key_*` / `kiri.gold.scd_type` | gold dimension | Dimension keys + SCD type. |
| `kiri.localization.names` | gold | Per-table localized name, e.g. `{sv: Försäljningsorder}`. |

**A gold fact** (from the `sales` example) needs `kind`, `grain`, and at least one
`measure`:

```yaml
- property: kiri.gold.kind
  value: fact
- property: kiri.gold.grain
  value: [order_id]
- property: kiri.gold.measures
  value:
  - name: gross_amount
    column: gross_amount
    aggregation: sum          # sum | count | count_distinct | min | max | avg
    additivity: additive      # additive | semi_additive | non_additive
- property: kiri.gold.dimension_refs
  value: [urn:kirimana:gold:dim_customer]
```

**Invariants** (any table) run after apply and fail the build on violation:

```yaml
- property: kiri.transformation.invariants
  value:
  - name: gross_amount_non_negative
    expression: "count_if(gross_amount < 0) = 0"
    expected: "true"
    severity: error
```

## Column level — `customProperties` on each `properties[]` entry

```yaml
properties:
- name: email
  logicalType: string
  physicalType: string
  description: Customer contact email.
  required: false
  unique: false
  primaryKey: false
  classification: confidential
  customProperties:
  - property: kiri.lineage.from_columns
    value: [urn:kirimana:bronze:customer::email]
  - property: kiri.pii.categories
    value: [email]
  - property: kiri.pii.direct_identifier
    value: true
```

ODCS-native column fields: `name`, `logicalType`, `physicalType`, `description`,
`required`, `unique`, `primaryKey`, `classification`, `businessName`, `examples`,
`tags`, `quality`.

| Property | Purpose |
|---|---|
| `kiri.lineage.from_columns` | Upstream column URN(s) this column derives from — powers column-level lineage + impact analysis. |
| `kiri.pii.categories` | PII category list, e.g. `[name]`, `[email]`, `[national_id]`, `[location]`, `[political_opinion]`. |
| `kiri.pii.direct_identifier` | bool — whether the column directly identifies a person. |
| `kiri.classification` | Column-level classification override. |
| `kiri.localization.names` | Per-column localized name, e.g. `{sv: E-post}`. |

> **PII ⇒ GDPR.** Any column marked with `kiri.pii.*` obliges its contract to
> declare `kiri.gdpr.lawful_basis` **and** `kiri.gdpr.purpose` (at table or layer
> level). `kiri contract validate --strict` and `kiri doctor` flag the omission —
> see the [mark-PII task](authoring-tasks.md).

## See also

- [Editing `kiri.yml`](authoring-kiri-yml.md) — the project config.
- [Authoring tasks](authoring-tasks.md) — add tables, switch technique, localize.
- [Validating your YAML](authoring-validation.md).
- [Silver-layer ways of working](silver-layer-ways-of-working.md) and
  [Naming, catalog and the schema registry](naming-catalog-and-schema-registry.md)
  for the concepts.
