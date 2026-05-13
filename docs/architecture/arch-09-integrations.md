# arch-09: Integrations

Kirimana is opinionated about contracts and conservative about reinvention. Where a mature open-source or open-standard tool already does a job well, Kirimana integrates rather than replaces. This document inventories the integration surfaces present in v0.9 preview — what they do, what maturity they carry, and where the integration boundary sits.

## dbt-core bridge

Kirimana treats dbt-core as the first **FrameworkAdapter** (arch-03). The bridge compiles silver and gold contracts into dbt project shape — `models/`, `sources.yml`, `schema.yml` — that dbt-core then executes against the platform. Kirimana does not fork dbt and does not replicate dbt's transformation engine. The contract is canonical; dbt is the execution surface.

Maturity: production-ready for the contract shapes documented in the bridge's compatibility matrix. Data Vault silver has an optional DataVault4dbt execution backend as a peer to the native compiler.

## OpenLineage

Every successful apply emits an OpenLineage `RunEvent` (arch-06). Column-level lineage rides as a facet. The transport is standard — HTTP or Kafka — and any OpenLineage-aware consumer (Marquez, DataHub, Astro Observe, platform-native UIs) ingests the events without Kirimana-specific code.

Maturity: production-ready. The lineage is generated from contract metadata, not inferred from query logs, so it is accurate by construction.

## MCP server

The Model Context Protocol server is Kirimana's external AI integration surface (arch-04). Any MCP client — Claude Code, Cursor, Continue, Zed, Claude Desktop — connects to a Kirimana project and uses a fixed tool set: `kirimana_check_policy`, `kirimana_validate_contract`, `kirimana_propose_change`, `kirimana_read_catalog`, `kirimana_search_assets`, `kirimana_get_asset`, and so on. Every call passes through the AI Gateway; every call is audited under the same event taxonomy as internal calls.

Maturity: production-ready as the protocol surface; the tool set is stable for the v0.9 functions and expands as new core operations land.

## Claude Code skills

Skills are Markdown-defined workflows that bottom out at the CLI or MCP tool layer (arch-04). They live in the same repository as the contracts they operate on, version with the code, and are discoverable to any agent that reads the `skills/` convention. Reference, project, and organization scopes layer in that order.

Maturity: production-ready as a convention. The reference skill library ships with the core; project and organization skills are authored by users.

## Airbyte ingest

Airbyte is the recommended-but-not-mandatory ingest backend. The integration is bidirectional in metadata, unidirectional in execution: Kirimana generates Airbyte source and connection configuration from source contracts, and reads Airbyte's destination state back into the catalog for verify (arch-02). Airbyte executes the ingest; Kirimana governs the contract.

Maturity: beta. Source onboarding is AI-assisted via the `kiri ingest new-source --interactive` flow (arch-04).

## Contract-PR GitHub Action

A GitHub Action runs `kiri validate` and `kiri plan` against PRs in a contract repository. On clean validation, it posts the plan diff as a PR comment. On merge to main, a companion workflow runs `kiri apply` against the dev cursor. Both are reference workflows in the public examples repo; teams typically copy and adapt rather than depend on them as a hosted action.

Maturity: production-ready as reference workflows; not a hosted GitHub Marketplace action.

## Slack bot

The Slack bot (`packages/slack_bot/`) listens for incident dispatches from the IncidentDispatcher protocol (alongside Jira, ServiceNow, and Zendesk integrations) and posts contract-change notifications to configured channels. It also exposes a minimal query interface — "what's the lineage for `urn:...`" — that bottoms out at the same MCP tool set.

Maturity: beta. Useful in production for notification; the query surface is light compared to the web app.

## Other surfaces

- **Incident dispatch to ITSM** — Jira, ServiceNow, Zendesk via the IncidentDispatcher protocol. Beta.
- **Catalog source pull** — Unity Catalog, Snowflake Horizon, Microsoft Purview as read-only CatalogSourceAdapter (arch-05). Beta.
- **DQX integration** — DQX integration is proposed (ADR 027); Great Expectations is the current quality engine. DQX will land as the Databricks-native option ahead of v1.0.
- **BimlFlex migration pack** — proposed paid integration for customers migrating from BimlFlex. Not in the open core.

## Integration boundary

A pattern runs through all of the above: Kirimana owns the contract, the plan, and the audit; the partner tool owns its native concern. dbt owns transformation execution. Airbyte owns connector logistics. OpenLineage owns the lineage wire format. Databricks owns workflow scheduling. Kirimana doesn't try to be any of those things, and the boundary is enforced through typed adapter protocols that make the integration testable and the conformance verifiable.

## Related ADRs

011, 017, 018, 019, 022, 027
