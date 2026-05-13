# ADR 022 — Catalog pull from Unity Catalog / Horizon / Purview

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** original ADR 0035 (archived)

## Context

Kirimana ships its own catalog and a push-side integration to external catalogs. Push assumes Kirimana is the source of truth and downstream tools mirror it — fine for greenfield deployments.

Real customers rarely start greenfield. A typical mid-market team has years of metadata invested in:

- **Databricks Unity Catalog** — table descriptions, owners, tags, column-level comments, lineage edges.
- **Snowflake Horizon** — sensitivity labels, data-quality monitors, classifications, masking policies.
- **Microsoft Purview** — data products, glossary bindings, scan-discovered classifications, lineage from multiple sources.

Asking that team to rebuild their catalog in Kirimana is a non-starter; the metadata investment is real money. They want Kirimana's value-add (per-contract AI policy, contract state machine, governance UI, PR-time linting) **on top of** what they already have. Today they cannot do that cleanly — manual CSV export is brittle, and the existing pluggable catalog backends are all Kirimana-managed stores.

Options considered:

1. **Tell those customers to start fresh.** Rejected — loses the deal.
2. **Make external catalogs a read+write Kirimana backend.** Rejected — bidirectional ownership invites split-brain; the external catalog is the source of truth for its own systems.
3. **Add a read-only `CatalogSourceAdapter` Protocol with idempotent sync semantics.** Accepted.

## Decision

Introduce a `CatalogSourceAdapter` Protocol — the read-only counterpart to `CatalogBackend`. Implementations stream normalised `ExternalAsset` objects into a Kirimana `CatalogBackend` on each sync run. Never write back.

Three v1 implementations:

| Source | Auth | Incremental | Pulled |
|---|---|---|---|
| Unity Catalog | Databricks OAuth2 / PAT | Yes (change-feed) | catalogs, schemas, tables, columns, tags, owners |
| Snowflake Horizon | Snowflake account + warehouse + role | Yes (`ACCOUNT_USAGE`) | databases, tables, columns, classifications, masking refs |
| Microsoft Purview | Entra service principal | No (full scan + diff) | data assets, glossary terms, classifications, lineage |

Sync runs via `kiri catalog pull --source <kind>` either one-shot or scheduled (cron, Kubernetes CronJob, CI). Each pull stamps `last_synced_at` and `source_state_hash` for forensic traceability and is idempotent.

**Conflict resolution: source wins on fields the source owns; Kirimana keeps fields the source does not expose.** Names, descriptions, owners, and classifications follow the source. Tags merge under namespaces (`unity:`, `kirimana:`). AI policy and governance customProperties stay Kirimana-owned. Per-field overrides are declared in `dca.yml`.

The CLI supports `--dry-run` (required posture for first-time pulls), `--limit`, and `--since` for incremental windows. Every pull emits one OpenLineage RunEvent for unified observability.

Out of v1: bidirectional sync, glossary hierarchy import, lineage-edge import, and external connector frameworks (OpenMetadata, DataHub). Revisit when a fourth source needs adding.

## Consequences

- **Positive:** Customers with existing catalogs onboard in days, not weeks. Kirimana's governance layers cleanly on top of imported assets. Symmetric with the push side — operators pick either or both. One Protocol means adding GCP Dataplex or AWS Glue later is one new file.
- **Negative / costs:** Three external API surfaces to maintain, each with its own auth, data model, and rate limits. Field-owner rules can surprise operators ("source overwrote my description") — `--dry-run` is the documented first posture. Each source needs separate vault entries.
- **Neutral:** Pull complements rather than replaces the contract-driven catalog import. Audit entries land in the same audit stream as MCP and AI gateway calls.

## Related

- See [arch-05-catalog-federation](../architecture/arch-05-catalog-federation.md)
- Related ADRs: 020 (hub-and-spoke catalog), 021 (federation resolver), 023 (column lineage)
