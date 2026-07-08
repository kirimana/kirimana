# Authoring tasks — how to make common changes

Recipes for the changes operators make most. Each is *goal → minimal diff →
verify*, on the `sales` example (`examples/sales/`). Field details are in
[Editing `kiri.yml`](authoring-kiri-yml.md) and
[Editing layer manifests](authoring-layer-manifests.md).

After any change, the baseline check is:

```bash
uv run kiri project validate --project .
uv run kiri layer validate   --project .
uv run kiri doctor           --project .
```

## Add a source

**Goal:** register a new source system so tables can ingest from it.

Add an entry under `sources:` in `kiri.yml`. Database source:

```yaml
sources:
  billing:
    host: sql.example.com
    database: billing_db
    port: 1433
    authentication: aad_az_cli
```

REST source:

```yaml
sources:
  rates_api:
    base_url: https://api.example.com/fx
    auth: ${vault:rates_api:token}
```

**Verify:** `kiri project validate`. Full walkthrough: [Adding a source](adding-a-source.md).

## Add a bronze table

**Goal:** land a new raw table in an existing domain.

Add a `schema[]` entry to `01-config/<domain>/bronze.yml`, pointing at its source:

```yaml
- name: order_line
  logicalType: table
  customProperties:
  - property: kiri.source_urn
    value: urn:kirimana:source:crm:order_line
  properties:
  - name: order_line_id
    logicalType: string
    physicalType: string
    description: Order line business key.
    required: true
    unique: true
    primaryKey: false
    classification: internal
  - name: order_id
    logicalType: string
    physicalType: string
    description: Parent order (joins to sales_order).
    required: true
    unique: false
    primaryKey: false
    classification: internal
```

**Verify:** `kiri layer validate`.

## Add a silver table

**Goal:** add a standardized silver table over a bronze feed (flat technique).

Add a `schema[]` entry to `silver.yml` with business keys and per-column lineage:

```yaml
- name: silver_order_line
  logicalType: table
  customProperties:
  - property: kiri.semantic.business_keys
    value: [order_line_id]
  - property: kiri.source_urn
    value: urn:kirimana:bronze:order_line
  properties:
  - name: order_line_id
    logicalType: string
    physicalType: string
    description: Order line business key.
    required: true
    unique: true
    primaryKey: true
    classification: internal
    customProperties:
    - property: kiri.lineage.from_columns
      value: [urn:kirimana:bronze:order_line::order_line_id]
```

For a data_vault silver, mark `kiri.dv.kind: hub|link|sat` instead of a flat table;
for scd2 add `kiri.silver.scd_type` + `kiri.silver.effective_from_column`. See
[Silver-layer ways of working](silver-layer-ways-of-working.md).

## Add a gold dimension or fact

**Goal:** expose an analytics-facing table.

**Dimension** — needs `kind`, business key, surrogate key, SCD type:

```yaml
- name: dim_product
  logicalType: table
  customProperties:
  - property: kiri.gold.kind
    value: dimension
  - property: kiri.gold.business_keys
    value: [product_id]
  - property: kiri.gold.surrogate_key_column
    value: product_key
  - property: kiri.gold.surrogate_key_strategy
    value: natural
  - property: kiri.gold.scd_type
    value: type_1
  - property: kiri.source_urn
    value: urn:kirimana:silver:silver_product
  properties: [ ... ]
```

**Fact** — needs `kind`, `grain`, and at least one `measure`:

```yaml
- name: fct_order_line
  logicalType: table
  customProperties:
  - property: kiri.gold.kind
    value: fact
  - property: kiri.gold.grain
    value: [order_line_id]
  - property: kiri.gold.measures
    value:
    - name: line_amount
      column: line_amount
      aggregation: sum
      additivity: additive
  - property: kiri.gold.dimension_refs
    value: [urn:kirimana:gold:dim_product, urn:kirimana:gold:dim_customer]
  - property: kiri.source_urn
    value: urn:kirimana:silver:silver_order_line
  properties: [ ... ]
```

**Verify:** `kiri plan --project .` lists the new gold contract; `kiri layer validate`.

## Switch silver technique

**Goal:** move from `flat` to another technique.

Edit `silver.technique` in `kiri.yml`:

```yaml
silver:
  technique: data_vault       # was: flat
  data_vault:
    execution_backend: native
```

The technique is locked after the first silver contract is created, so this is a
deliberate, project-wide change; the silver manifest's table shapes must match the
new technique (hub/link/sat for data_vault, dim/fact naming for kimball). Reconcile
the silver manifest, then `kiri layer validate`.

## Set up a localized (Swedish) twin — without a collision

**Goal:** generate a localized twin of the gold serving schema.

Enable localization at the gold layer level, and give **canonical** tables their
localized names — do **not** hand-declare a table into the generated twin schema
(that triggers the ADR 0133 §4.2 twin-schema collision):

```yaml
# gold.yml — layer level
customProperties:
- property: kiri.layer
  value: gold
- property: kiri.localization.default_locale
  value: en
- property: kiri.localization.locales
  value: [sv]
- property: kiri.localization.schema_pattern
  value: "{schema}_{locale}"
- property: kiri.localization.on_missing
  value: fail
```

```yaml
# gold.yml — a canonical table opts into a twin via names (NOT target_schema)
- name: dim_customer
  logicalType: table
  customProperties:
  - property: kiri.target_schema
    value: serving
  - property: kiri.localization.names
    value: {sv: Kund}
  properties:
  - name: full_name
    ...
    customProperties:
    - property: kiri.localization.names
      value: {sv: Namn}
```

**Verify:** `kiri layer validate` must be green. If it reports a *twin-schema
collision*, a table is declaring `kiri.target_schema: serving_sv` while the twin
also renders `serving_sv` — move that table to a distinct schema or drop the
auto-twin.

## Configure targets and secrets

**Goal:** add a production Databricks target without leaking secrets.

```yaml
# kiri.yml
targets:
  prod:
    adapter: databricks
    host: https://adb-xxxx.azuredatabricks.net
    http_path: /sql/1.0/warehouses/abc123
    warehouse_id: abc123
    catalog: sales_prod
    auth_type: azure-sp
    token: ${vault:databricks/prod:token}
vault:
  kind: env
```

Store the secret out of band (env var `KIRIMANA_...` for `vault.kind: env`), never
in YAML. **Verify:** `kiri project validate`; connectivity via
`kiri databricks health --target prod`. See
[Targets, profiles and secrets](targets-profiles-and-secrets.md).

## Drive catalog names by pattern + preserve verbatim names

**Goal:** render per-layer catalogs from one pattern and keep source names exact.

```yaml
# kiri.yml
naming:
  preserve_verbatim: true
  catalog_pattern: "{env}_dp_{domain}_{layer}"   # e.g. dev_dp_sales_bronze
```

With a pattern set, leave `targets.<env>.catalog` blank in the layer manifests.
Only `{env}`, `{domain}`, `{layer}` are valid tokens. **Verify:** `kiri layer
validate` (an unknown token fails at parse time).

## Mark PII and satisfy GDPR

**Goal:** flag a personal-data column correctly.

Mark the column, and ensure its table (or layer) declares lawful basis + purpose:

```yaml
# on the column
- name: email
  ...
  customProperties:
  - property: kiri.pii.categories
    value: [email]
  - property: kiri.pii.direct_identifier
    value: true
```

```yaml
# on the table (or the layer) that owns PII
customProperties:
- property: kiri.classification
  value: confidential
- property: kiri.gdpr.lawful_basis
  value: contract          # e.g. contract | consent | legal_obligation | public_task | ...
- property: kiri.gdpr.purpose
  value: Fulfil and account for customer sales orders.
```

**Verify:** `kiri contract validate <file> --strict` and `kiri doctor` — both flag
PII without a lawful basis/purpose. Field semantics:
`https://github.com/kirimana/kirimana/blob/main/docs/contracts/metadata-master.md`.

## Add business-rule invariants

**Goal:** fail the build if a business rule is violated.

```yaml
# on the table
customProperties:
- property: kiri.transformation.invariants
  value:
  - name: gross_amount_non_negative
    expression: "count_if(gross_amount < 0) = 0"
    expected: "true"
    severity: error
```

**Verify:** `kiri layer validate` (structure); the check runs post-apply (ADR 0102).

## Configure reconcile (migrations)

**Goal:** cross-engine reconcile a migrated table against its source of truth.

```yaml
# kiri.yml
reconcile:
  pk_check_mode: sample
  pk_sample_size: 1000
  tolerances:
    row_count: {pass_max: 0.005, unit: percent}
    null_rate: {pass_max: 0.01, unit: percent}
  tables:
    - {source: crm.sales_order, target: sales_demo.silver.silver_sales_order, primary_key: [order_id]}
```

**Verify:** `kiri reconcile --source-target <src> --target-target <tgt>`. See
[Backfill and reprocess](backfill-and-reprocess.md).
