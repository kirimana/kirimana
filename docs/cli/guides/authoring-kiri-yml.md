# Editing `kiri.yml` — the project config reference

`kiri.yml` sits at the repo root and configures the whole project: identity,
adapter, connection targets, silver technique, naming, secrets, and governance.
This page documents every block. Field types come from the canonical
`ProjectConfig` model; where a field is an enum, only the listed values validate.

Validate any change with:

```bash
uv run kiri project validate --project .
```

The complete `sales` example is at `examples/sales/kiri.yml`.

## Minimal project

Only `name` and `adapter` are strictly required; a usable project also needs at
least one target and a `default_target`:

```yaml
name: sales-demo
version: 0.1.0
adapter: localduckdb
default_target: localduckdb
targets:
  localduckdb:
    path: ./.kiri/warehouse.duckdb
    bronze_root: ./.kiri/bronze
    schema: bronze
silver:
  technique: flat
vault:
  kind: env
```

## Identity

| Field | Required | Type / values | Notes |
|---|---|---|---|
| `name` | **yes** | string | Project name. Also the default job-name prefix (see `orchestration`). |
| `version` | no | string (default `0.1.0`) | Project version. |
| `adapter` | **yes** | string | Runtime adapter, e.g. `localduckdb`, `databricks`. Exercised runtimes today are DuckDB + Databricks (ADR 0130). |
| `layout` | no | `per_domain` \| `per_source` (default `per_domain`) | How source/contract files are organised on disk. |
| `mode` | no | `contract` \| `auto` \| `hybrid` (default `contract`) | DPA authoring mode (ADR 0078). `contract` = you own the YAML. |
| `default_target` | no | string | Which `targets` entry to use when `--target` is omitted. |

## `targets` — connection profiles

A map of named environments (e.g. `dev`, `test`, `prod`, or `localduckdb`). Each
entry is a free-form profile the adapter reads; fields depend on the adapter.

```yaml
targets:
  localduckdb:                      # local dev — no cloud
    path: ./.kiri/warehouse.duckdb
    bronze_root: ./.kiri/bronze
    schema: bronze
  prod:                             # Databricks
    adapter: databricks             # per-target adapter override (optional)
    host: https://adb-xxxx.azuredatabricks.net
    http_path: /sql/1.0/warehouses/abc123
    warehouse_id: abc123
    catalog: sales_prod
    schema: bronze
    governance_schema: sales_governance
    bronze_root: ./.kiri/bronze
    auth_type: azure-sp             # adapter-specific; see below
    token: ${vault:databricks/prod:token}
```

Common target fields:

| Field | Applies to | Notes |
|---|---|---|
| `path` | localduckdb | DuckDB file path. |
| `host` / `http_path` / `warehouse_id` | databricks | SQL warehouse endpoint. |
| `catalog` / `schema` | databricks | Connection defaults; per-model routing may override (see `naming.catalog_pattern`). |
| `governance_schema` | all | Schema for governance/audit objects. |
| `bronze_root` | all | Local staging root for landed files. |
| `auth_type` | databricks | Authentication mode. Adapter-specific; common values: `azure-sp` (Azure-AD service principal), `oauth_m2m` (OAuth machine-to-machine), or a `token`/PAT supplied via `${vault:}`. |
| `token` | databricks | Secret reference (`${vault:...}`), never a literal. |
| `ingest_volume` | databricks | UC Volume used to stage landing files for `read_files()`. |
| `landing_storage` | databricks | Cloud landing block (ADR 0046) for production. |
| `libraries` / `project_volume` / `new_cluster` | databricks | Scheduled-Job compute for `kiri flows sync` (ADR 0052/0053): the wheel + synced project + job cluster. |

Secrets in any target use `${vault:<id>:<key>}` (see `vault`).

## `sources` — where raw data comes from

A map of named source systems. Database sources (e.g. SQL Server) declare a
connection; REST sources declare an endpoint. Authentication follows the same
no-plaintext rule.

```yaml
sources:
  crm:                              # a database source
    host: sql.example.com
    database: crm_db
    port: 1433
    authentication: aad_az_cli      # Entra token from the ambient az session
  catalog_api:                      # a REST source
    base_url: https://api.example.com/v1
    auth: ${vault:catalog_api:token}
```

For the full add-a-source walkthrough see [Adding a source](adding-a-source.md).

## `silver` — the silver technique

```yaml
silver:
  technique: flat                   # flat | data_vault | kimball_dimensional | scd2_curated
  emit_curated_history: true        # meaningful for scd2_curated
  data_vault:                       # only when technique=data_vault
    execution_backend: native       # native | datavault4dbt
```

| Field | Type / values | Notes |
|---|---|---|
| `technique` | **required** — `flat` \| `data_vault` \| `kimball_dimensional` \| `scd2_curated` | Project-wide silver strategy (ADR 0013). Locked after the first silver contract. |
| `emit_curated_history` | bool (default `true`) | For `scd2_curated`: whether to emit the `__curated` SCD2-history peer alongside the standardized table. |
| `data_vault.execution_backend` | `native` \| `datavault4dbt` | Only valid when `technique=data_vault` (ADR 0062). |

To change technique later, see the task [Switch silver technique](authoring-tasks.md).

## `naming` — object names and catalog pattern

```yaml
naming:
  preserve_verbatim: true                     # byte-for-byte declared names (ADR 0128)
  catalog_pattern: "{env}_dp_{domain}_{layer}" # renders each layer's catalog (ADR 0135)
```

| Field | Type | Notes |
|---|---|---|
| `preserve_verbatim` | bool | When true, physical names equal declared names exactly (case + diacritics). Required for exact-match migrations. |
| `catalog_pattern` | string | Template for per-layer catalog names. Closed token set: `{env}`, `{domain}`, `{layer}` — any other `{token}` is rejected at parse time. Literal text between tokens is emitted verbatim. When set, layer manifests leave `targets.<env>.catalog` blank. |
| `medallion` | block | Per-layer schema-name overrides (bronze/silver/gold, plus named silver zones — ADR 0048/0137). |
| `dv` / `kimball` | blocks | Technique-specific naming (hub/link/sat suffixes; dim/fact prefixes). |

Concepts and the schema registry: [Naming, catalog and the schema registry](naming-catalog-and-schema-registry.md).

## `orchestration` — scheduled-job naming

```yaml
orchestration:
  job_name_prefix: dp_sales_        # compiled jobs become dp_sales_<flow>
```

| Field | Type | Notes |
|---|---|---|
| `job_name_prefix` | string \| omit | Prefix for compiled Databricks Workflows job names (ADR 0141). Omit to accept the neutral project-slug default (`<name>_`). |

## `vault` — secret resolution

```yaml
vault:
  kind: env                         # env | azure | aws
  prefix: ""                        # optional id prefix, e.g. "kirimana/"
  vault_url: null                   # for kind=azure/aws
```

`${vault:<id>:<key>}` references anywhere in the project resolve through this
backend (ADR 0011). `kind: env` reads `$KIRIMANA_*` environment variables.

## `governance` and `domains`

```yaml
governance:
  mode: light                       # light | enterprise (approval tiers, ADR 0044)
domains:
  sales:
    owners: [sales-data@example.com]
    default_classification: internal
    retention_days: 3650
    approval_policy:
      default_approvers: 1
      classification_raise_approvers: 2
      cross_domain_consumption_approvers: 2
```

| Block | Field | Notes |
|---|---|---|
| `governance` | `mode` | `light` (single-approver, default) or `enterprise` (two-approver on destructive actions). |
| `domains.<name>` | `owners` | Domain owner emails. |
| | `default_classification` | Default `kiri.classification` for the domain. |
| | `retention_days` | Default retention. |
| | `approval_policy.*` | Approver counts per action class (ADR 0025/0030). |

## Other blocks (optional)

| Block | Purpose | Reference |
|---|---|---|
| `semantic_layer` | Regenerate BI-tool semantic-layer files on apply (ADR 0036). | [Consuming gold for BI](consuming-gold-for-bi.md) |
| `audit` / `compliance` | Audit-log + compliance-report settings (ADR 0037/0032). | [Security and compliance](security-and-compliance.md) |
| `incidents` | ITSM incident dispatch (ADR 0038). | — |
| `reconcile` | Post-migration cross-engine reconcile (ADR 0054). | [Backfill and reprocess](backfill-and-reprocess.md) |
| `sizing` | Compute-sizing hints. | — |
| `project_chunks` | Chunk-awareness for large projects (ADR 0110). | — |
| `allow_legacy_source_fallback` | Permit uncontracted ingest during migration (emits a warning). | — |

## See also

- [Editing layer manifests](authoring-layer-manifests.md) — the bronze/silver/gold files.
- [Authoring tasks](authoring-tasks.md) — step-by-step changes.
- [Validating your YAML](authoring-validation.md).
