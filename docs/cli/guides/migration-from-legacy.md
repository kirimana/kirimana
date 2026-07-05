# Migrating from a legacy warehouse

Most teams don't start from zero — they start from a warehouse full of tables and stored procedures that nobody fully remembers. Kirimana's migration surface is a funnel: get a governed scaffold out of the live system, translate the logic, verify the target, and prove the migrated data matches the source. This guide covers the **open** CLI surface and notes where the licensed Migration Pack plugs in.

## The funnel at a glance

```
Live legacy DW
   │  kiri discover           → scaffold contracts from live introspection
   ▼
Governed scaffold
   │  kiri migrate analyze    → sources/*.yml from MSSQL system views
   │  kiri migrate translate-tsql / sp → T-SQL → Spark SQL
   ▼
Built target
   │  kiri migrate verify     → Databricks pre-deploy infra checks
   │  kiri migrate verify-names → verbatim name-preservation gate
   ▼
Validated cutover
      kiri reconcile          → source-vs-target row/PK/null/date checks
```

## 1. Discover — a scaffold from a live warehouse

`kiri discover` introspects a live data warehouse **read-only** and emits a Kirimana scaffold with heuristic medallion and classification suggestions. It's the bottom-up entry point for a system that has tables but no metadata.

```bash
kiri discover --target sqlserver --conn "${vault:legacy/dw:conn}" --output scaffold
```

Useful knobs:

- `--schemas dbo,sales` — scan only certain schemas; `--max-tables N` for a smoke run.
- `--known-source 'erp:dbo.*'` — pin a known source grouping, overriding the FK-derived guess (repeatable).
- `--with-ai --max-cost 5.00` — opt into AI-augmented descriptions and classification with a hard USD spend cap (AI is off by default; the cap is required when it's on).
- `--layout layer --domain sales` — emit per-`(domain, layer)` manifests instead of the legacy per-table layout.

Row data never leaves your perimeter unless you explicitly ask: sample-row egress requires both `--probe-sample-data N` and `--confirm-data-egress`, and PII-suspect columns are excluded from any sample.

> Note: this is distinct from `kiri ingest discover`, which probes ingest-connector schemas. `kiri discover` scaffolds a whole project from a live DW.

## 2. Analyze and translate the logic

Introspect an MSSQL source into `sources/*.yml` (read-only against `sys.*` system views). Encrypted procedures are forced `out_of_scope`; low-confidence classifications carry a `# REVIEW:` flag so nothing is silently trusted:

```bash
kiri migrate analyze --source-target mssql --output sources/mssql.yml
```

Translate T-SQL to Spark SQL. For small, trivially translatable bodies, the deterministic translator does it with **no AI calls** — residual constructs are surfaced as warnings rather than guessed at:

```bash
kiri migrate translate-tsql -i ./proc.sql -o ./proc.spark.sql
```

For a full stored procedure, `kiri migrate sp` classifies and translates one procedure end to end. Apply the layer-policy validator across the generated models with `kiri migrate lint-models`.

## 3. Verify the target before cutover

Run pre-deploy infrastructure checks against the Databricks target — catalog, schemas, the `_kiri_meta` schema, external location, storage credential, and SQL warehouse. It refuses to print `ALL PASS` unless every check passed:

```bash
kiri migrate verify --target prod
```

Enforce verbatim name preservation — that migrated tables and columns keep their legacy names exactly (NAME-001 table, NAME-002 column):

```bash
kiri migrate verify-names --target prod
```

If your contracts declare foreign keys, emit the constraint DDL for every silver contract to apply on the target:

```bash
kiri migrate emit-fk-ddl --target prod
```

## 4. Reconcile — prove the data matches

After the target is built, `kiri reconcile` runs post-migration data validation: source-vs-target row counts, primary-key uniqueness, null rates, and date bounds. It handles SCD2 shapes and can run in a `locked` report mode that omits raw values for sensitive data.

```bash
kiri reconcile \
  --source-target mssql --target-target databricks \
  --source-table dbo.customer --target-table sales.customer \
  --primary-key customer_id \
  --columns email,created_at --date-columns created_at \
  --report-mode locked --format markdown -o ./reconciliation/customer.md
```

A green reconcile report is your cutover evidence — the artefact you show the business to sign off that the new warehouse is faithful to the old one.

## Where the Migration Pack plugs in

Everything above is the **open surface**. The **Migration Pack** is a separate, licensed component for accelerating migrations from specific legacy tooling — it plugs in behind a licence gate and adds source-specific readers and translators. Its verbs live under the same `kiri migrate` group; for example:

```bash
kiri migrate from-bimlflex   # requires the Migration Pack licence
```

Without the pack installed, that verb reports the licence requirement and the open funnel above still gets you a governed, reconciled migration by hand.

## Related reference

- [Migration](../migration.md)
- [Databricks and adapters](databricks-and-adapters.md)
- [Data quality and invariants](data-quality-and-invariants.md)
