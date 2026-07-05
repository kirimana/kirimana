# Adding a source

A *source* declares where data comes from and how Kirimana lands it. This guide covers the ingest engines available, the two authoring paths (AI-assisted vs manual), and how credentials are wired.

Kirimana never silently picks an ingest engine for you — you always choose. Start by listing what's available.

## List the ingest modes

```bash
kiri ingest modes
```

This prints the read-only catalog of every mode, its ingestion family, and whether it needs credentials.

### Which modes are live vs planned

| Mode | Family | Status | Notes |
|---|---|---|---|
| `landing_zone` | push | **Live** | An external system writes files into a watched path; Kirimana converts and registers them. No credentials held by Kirimana. |
| `rest_api` | pull | **Live** | Kirimana fetches from an HTTP endpoint into the landing zone. Auth in vault. |
| `database` | pull | **Live** | Kirimana queries a source database (Postgres / MySQL / MSSQL / Oracle / SQLite) and lands the result. Credentials in vault. |
| `soap` | pull | **Live** | Kirimana POSTs a SOAP envelope and lands the XML response. Same auth surface as REST. |
| `airbyte` | pull | **Live** | An Airbyte Connection drives the fetch; Kirimana reads from the destination path. Connector, retry, and schema-drift are owned by Airbyte. |
| `fixture_file` | generated | **Live (dev only)** | Local CSV/JSON/XML in `data/raw`. Demos and tests; never production. |
| `dlt` | pull | **Planned** | Route via `rest_api` or `database` today. |
| `debezium` | push | **Planned** | Land change events via `landing_zone` today; wire Debezium externally. |
| `stream` | push | **Planned** | Kafka / Kinesis / Event Hubs / Pulsar. Route via `landing_zone`, `rest_api`, or `database` today. |

Always confirm the current catalog with `kiri ingest modes` — it is the source of truth for your installed version.

## Pull vs push families

- **Pull** — Kirimana actively reaches out and fetches (`rest_api`, `database`, `soap`, `airbyte`). Kirimana holds the credentials (in vault) and controls cadence.
- **Push** — an external system delivers data to Kirimana (`landing_zone`, and the planned `debezium` / `stream`). Kirimana holds no source credentials; it watches a path and registers what arrives.

Pull sources trigger an implicit fetch into the landing zone before ingestion; push sources rely on the external writer having already delivered.

## Path A: AI-assisted authoring

`kiri ingest new-source` runs a backend-agnostic onboarding flow. If you omit `--backend`, Kiri asks which engine to use and routes from there. Nothing is written to disk without your confirmation.

```bash
# Let Kiri ask which engine to use (recommended for new users)
kiri ingest new-source

# Or route straight to a chosen engine
kiri ingest new-source --backend rest_api
kiri ingest new-source --backend database
kiri ingest new-source --backend airbyte
kiri ingest new-source --backend landing_zone
```

For an Airbyte source, you can probe a connector's schema before committing:

```bash
kiri ingest discover
```

(Note: this is the Airbyte connector probe — distinct from the top-level `kiri discover`, which scaffolds a whole project from a live warehouse.)

## Path B: Manual authoring

Every field is operator-supplied — deterministic, no AI. Same output shape as the AI flow.

**Scaffold** a source YAML:

```bash
kiri source scaffold crm \
  --owner data-team@example.com \
  --domain sales \
  --classification internal \
  --mode database \
  --tables accounts,contacts
```

**Introspect** a reachable source to populate a draft from a real sample:

```bash
kiri source introspect crm accounts \
  --owner data-team@example.com \
  --domain sales \
  --mode database
```

**Validate**, then **commit** atomically:

```bash
kiri source validate --content sources/crm.yml
kiri source commit --target sources/crm.yml --content sources/crm.yml
```

`kiri source commit` re-validates, writes via an atomic replace, keeps a `.bak` of the prior file, and supports an optimistic SHA-256 lock (`--checksum`) so a concurrent edit can't be silently overwritten.

## Credentials — always via vault

Source credentials are **never** written into YAML in plaintext. You reference them with `${vault:<id>:<key>}` in the source config, and store the actual value in the project vault:

```bash
kiri vault set airbyte/salesforce --key password
kiri vault list        # redacted inventory — key names only
```

In the source YAML the value appears only as a reference — for example `${vault:airbyte/salesforce:password}`. CI fails on any detected plaintext secret. See [targets, profiles & secrets](targets-profiles-and-secrets.md) for the full vault reference syntax.

## Trigger the ingest

Once a source is committed, drive its backend explicitly:

```bash
# Airbyte-backed sources
kiri ingest sync --backend airbyte --source crm

# Native path (runs through kiri apply)
kiri ingest sync --backend native --source crm

# Landing-zone / file scan → Parquet → bronze views
kiri ingest bronze --source crm
```

`--backend` is required — Kirimana never defaults it. Add `--dry-run` to preview the plan without calling the backend. For the full pipeline (ingest → contracts → dbt build) in one step, use `kiri apply`.

## Related reference

- [sources & ingestion](../sources-ingestion.md) — every `kiri source` and `kiri ingest` command
- [secrets](../secrets.md) — `kiri vault` and the `${vault:...}` reference syntax
- [contracts & schema](../contracts-schema.md) — turn a source into a contract with `kiri contract new`
- [build, plan & apply](../build-plan-apply.md) — `kiri apply`, `kiri plan`
