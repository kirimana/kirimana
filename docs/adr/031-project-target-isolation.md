# ADR 031 — Project-to-target namespace isolation

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** original ADR 0014 (archived)

## Context

A Kirimana project commits to a single silver modelling technique (Data Vault, Kimball, or flat) for its entire lifetime. Organisations with heterogeneous modelling needs must run multiple Kirimana projects.

A natural follow-on question: can those projects share a physical target — one Databricks workspace, one DuckDB host, one Fabric warehouse — without stepping on each other? The answer matters because the typical enterprise has *one* Databricks workspace per business unit but *many* governance domains within it.

The risks of running multiple projects in one undifferentiated namespace are real and severe:

- **Schema collision.** Two projects both writing `silver.customer` — last-writer-wins, opposite shapes (DV vs Kimball) corrupting the table.
- **Bronze fetch duplication.** Two projects both ingesting the same source double-pay the budget and race on landing-zone files.
- **Concurrent-write contention.** Delta MERGE, DuckDB file locks, Unity transactions all assume a single logical writer.
- **Catalog URN collision.** Two projects registering the same URN cause one to overwrite the other silently.
- **Classification erosion.** Project A's PII classification on a table can be lost if project B writes over the same physical table.

Kirimana's job is to make these structurally impossible, not to catch them in CI.

## Decision

### The catalog is the project boundary

A Kirimana project maps 1:1 to **one catalog on the target platform**. Schemas, tables, views, and lineage entries live inside that catalog and belong exclusively to that project. The catalog is the strong boundary; the workspace, server, or account above it is shared freely.

| Target platform | Shared freely | One per project |
|---|---|---|
| Databricks | Workspace, account | Unity Catalog |
| Fabric | Capacity, workspace | Lakehouse |
| Trino + Iceberg | Cluster, Polaris instance | Catalog |
| DuckDB | Host, file system | Database file |
| MSSQL | Server, instance | Database |

### Three structural rules

1. **No cross-catalog writes from a project.** A project may *read* from another catalog (with explicit cross-project contract reference), but never write outside its own catalog. Enforced at plan-time, not at apply-time.
2. **Bronze landing zone is shared, bronze tables are not.** Multiple projects can land raw bytes in the same landing zone for storage efficiency, but each project's bronze tables live in its own catalog.
3. **Catalog name derives from the project URN.** `<project_urn>_<environment>` (e.g., `kirimana_acme_sales_prod`) — naming is deterministic, not operator-chosen, eliminating typos.

## Consequences

- **Positive:** Multiple Kirimana projects coexist safely in one workspace. Failure modes (write collision, catalog collision) become impossible by construction.
- **Negative:** Enterprises with many small projects accumulate catalogs. Catalog count grows with project count; some platforms have soft limits to monitor.
- **Neutral:** Cross-project data sharing requires explicit federation lookups, which is a feature not a bug — it makes contract dependencies visible.

## Related

- See [arch-05-catalog-federation.md](../architecture/arch-05-catalog-federation.md)
- Related ADRs: 007 (lifecycle), 020 (catalog), 021 (federation), 032 (multi-environment)
