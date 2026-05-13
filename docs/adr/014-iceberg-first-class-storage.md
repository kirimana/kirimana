# ADR 014 — Iceberg as a first-class storage primitive

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** originally ADR 0024 (archived)

## Context

Kirimana's initial lakehouse story ran end-to-end on Delta Lake — `delta-rs` for landing-zone writes, a `DeltaDialect` for rendering, all platform adapters landing data as Delta tables. Delta is mature, widely adopted, and the bronze layout was designed around it.

The format landscape has shifted: Snowflake ships native Iceberg table support; Microsoft Fabric adopted Iceberg alongside Delta in OneLake; AWS Lake Formation and Glue default to Iceberg for new tables; Google BigLake uses Iceberg as its federation primitive; Databricks shipped Unity Catalog-managed Iceberg tables as a first-class citizen. Apache Polaris, Nessie, and the Unity Catalog REST API give Iceberg an open catalog layer Delta never quite had.

The multi-cloud / multi-platform enterprise segment — the one the federated contract library aims at — increasingly runs Iceberg as the cross-platform neutral format. A "Tabular-style" architecture (one Iceberg catalog consumed by N engines) is now plausible where it was not in 2024.

Three positions were considered: stay Delta-only (rejected — breaks the portability promise); replace Delta with Iceberg (rejected — Delta is mature and existing customers are legitimately on Delta); ship Iceberg as a peer to Delta (accepted).

## Decision

**Add `IcebergPlatformAdapter` and `IcebergDialect` as peers to the existing Delta + DuckDB stack. Contracts stay format-agnostic; the target's adapter decides which format materialises.**

| Platform adapter | Default dialect | Default format |
|---|---|---|
| `localduckdb` | `DuckDbDialect` | DuckDB native |
| `databricks` | `DeltaDialect` | Delta Lake |
| `fabric_lakehouse` *(future)* | `DeltaDialect` | Delta (OneLake) |
| `iceberg_s3` *(new)* | `IcebergDialect` | Iceberg (REST catalog) |
| `iceberg_glue` *(new)* | `IcebergDialect` | Iceberg (Glue catalog) |
| `iceberg_polaris` *(new)* | `IcebergDialect` | Iceberg (Polaris) |
| `iceberg_unity` *(new)* | `IcebergDialect` | Iceberg (Unity Catalog) |

The adapter's profile selects the catalog backend (REST, Glue, Polaris, Unity Catalog); the dialect is shared because Iceberg's SQL surface is catalog-agnostic.

### `pyiceberg` as reference implementation

`pyiceberg` (Apache project, used by Tabular and Polaris) plays the role `delta-rs` plays for Delta: catalog abstraction, table metadata read/write without a Spark dependency, snapshot and schema-evolution tracking compatible with every Iceberg-reading engine, partition spec and sort-order primitives the SQL dialect emits. No JVM required; runs in the Kirimana CLI process. CI-testable with the REST-catalog fixture that ships with the project.

### IcebergDialect — parallel to DeltaDialect

Same contract as `DeltaDialect`: identifier quoting, hash expression, MERGE INTO syntax, partition spec with transforms (`day(ts)`, `bucket(16, id)`) — the first Kirimana surface that really needs per-dialect partition DSL — and `time_travel_as_of_snapshot` for point-in-time queries via Iceberg snapshot IDs.

### Bronze layout stays format-agnostic

The landing-zone bronze layout (landing then raw then materialised table) was specified format-agnostically. The final materialisation step is adapter-pluggable: LocalDuckDB materialises as DuckDB table, Databricks as Delta, Iceberg adapter as Iceberg with snapshot isolation. Same ingestion pipeline, different final-write target.

### Cross-format contract materialisation

A project with `targets:` declaring both Delta and Iceberg materialises the same contract into both — this is the multi-cloud exit option the architecture promises. Release status displays drift across formats.

### Catalog interoperability

Kirimana's governance catalog is unaffected. Iceberg's engine-level catalog (Polaris, Glue, Nessie, REST) is distinct from the governance catalog — the governance catalog tracks contracts, classifications, ownership; the Iceberg catalog tracks table metadata, snapshots, schema evolution. Kirimana populates both via the adapter. A follow-up task pushes classifications from the governance catalog into Iceberg-catalog table properties so engines see Kirimana's governance natively.

## Consequences

- **Positive:** Cross-format portability — a Kirimana project migrates from Delta to Iceberg by swapping target adapter; contracts, ownership, AI policy, and lineage survive unchanged; enterprise neutral-format adoption (Snowflake + Databricks + Trino reading the same tables) gets Kirimana as the governance layer; federation becomes load-bearing across formats; Iceberg snapshot IDs integrate naturally with release-SHA stamping for deterministic point-in-time queries.
- **Negative / costs:** Dual-format maintenance — every SQL emitter now needs per-format dialect branching (mitigated because format is already factored out of the planner via the dialect abstraction); `pyiceberg` is younger than `delta-rs`, so some write-path corner cases lag Spark-based equivalents (accepted — Kirimana does not need the full Spark-Iceberg surface, only the slice contracts emit); catalog backend diversity (REST / Glue / Polaris / Unity / Nessie) means Kirimana writes against the `pyiceberg.Catalog` interface, not each backend.
- **Neutral:** Iceberg adoption reads Kirimana's value exactly the way Delta does — classifications, AI policy, audit, governance UI, MCP all work identically. The format-agnostic metadata is the invariant.

## Related

- See [arch-03-adapters](../architecture/arch-03-adapters.md)
- Related ADRs: 011, 013
