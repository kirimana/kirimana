# ADR 017 — MCP as external AI surface

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** original ADR 0023 (archived)

## Context

Kirimana exposes four user-facing surfaces: CLI, web UI, AI gateway, and the MCP server. Platform-native AI assistants — Databricks Assistant, Snowflake Cortex Copilot, Microsoft Fabric Copilot, GitHub Copilot — each need a way to know what Kirimana knows about a given dataset (contract, ownership, classification, lineage, AI policy).

Three options were considered:

1. **Ignore external assistants.** Rejected — undermines AI policy enforcement the moment a user opens a non-Kirimana assistant.
2. **Per-vendor plugins.** Build a Databricks Assistant plugin, a Cortex plugin, a Fabric plugin. Rejected — N plugins against N vendor interfaces, each breaking on the other's upgrades.
3. **Model Context Protocol (MCP) as the single integration surface.** One server implementation; any MCP-speaking client integrates by configuration, not by code.

This ADR picks option 3 and defines what the MCP server must expose to be substantive — not merely a catalog read-view.

## Decision

The MCP server in `packages/mcp_server` is Kirimana's authoritative external AI integration surface. Kirimana does not build vendor-specific AI plugins.

The server exposes six resource and tool types mapped to Kirimana's value propositions:

| Surface | Shape | Purpose |
|---|---|---|
| `kirimana://contract/{name}` | Resource | Read full ODCS contract |
| `kirimana://classification/{fq_name}` | Resource | Map warehouse name → classification |
| `kirimana://lineage/impact/{contract}` | Resource | Impact analysis |
| `kirimana://ai-policy/{contract}` | Resource | Read-only AI policy view |
| `kirimana://release-status` | Resource | Current release SHA in prod |
| `kirimana.ai.check_policy(contract, operation)` | Tool | Runtime enforcement: allow / deny / transform |

The `check_policy` tool is the load-bearing addition — it turns external assistants from "information consumers" into "policy-respecting agents". It delegates to the same `AIGateway.check_policy()` used internally, so external clients receive identical enforcement and audit entries land in one stream.

Every MCP resource read and tool call writes an audit entry carrying the caller identifier (e.g. `mcp.databricks-assistant`, `mcp.claude-code`).

This ADR explicitly does NOT:
- Enforce policy against non-MCP assistants — enforcement is cooperative.
- Move data — the MCP server returns metadata only.
- Build vendor-specific panels — vendors build their own UI on top of MCP.
- Allow MCP write tools to bypass git-PR review.

## Consequences

- **Positive:** One integration, many clients. AI policy becomes enforceable on third-party assistants that opt in. Centralised audit trail across vendors. MCP is open-spec, not single-vendor lock-in.
- **Negative / costs:** Adoption depends on assistants implementing MCP client mode. Policy enforcement is cooperative — non-cooperative assistants proceed without governance. Audit-log volume grows; sampling may be needed for chatty read-only paths.
- **Neutral:** Existing surfaces unchanged. New resources are additive to the current MCP scaffold.

## Related

- See [arch-04-ai-gateway](../architecture/arch-04-ai-gateway.md)
- Related ADRs: 016 (AI Gateway trust ladder), 018 (Claude skills), 021 (federation)
