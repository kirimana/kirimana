# ADR 021 — Federation resolver for cross-project contracts

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** original ADR 0066 (archived)

## Context

Cross-project federation has become load-bearing across the architecture:

- Project / target isolation declared that cross-project access must go through federation, not direct SQL.
- Iceberg lakehouse adoption makes federation load-bearing across storage formats, not only across environments.
- Column-level lineage cross-project lookup requires a federation index.
- Multi-environment CI/CD deferred cross-project atomic release without defined semantics.

Each surrounding ADR did the right thing for its own scope, but the federation **contract itself** — what is published, how clients fetch it, what happens when it is down — had never been written. Implementations were making implicit choices that might or might not match consumer expectations.

This ADR locks the contract.

## Decision

Federation is a documented public protocol. Every Kirimana project publishes a versioned **federation export**; every consumer (other Kirimana projects, MCP-connected AI assistants, web UIs, lint tools) reads through one of three transport flavours of the same resolver API.

**1. Federation export.** A deterministic, JSON-serialised snapshot under `.kirimana/federation/export.json` containing `schema_version`, `project_urn`, `release_sha`, `catalog_etag`, and per-contract entries with URN, domain, owner, classification, AI policy, schema columns with lineage edges (`from_columns`), reporting goals, and release state. The export excludes SQL bodies, internal customProperties, audit log entries, quality results, and secrets. Regenerated on every successful apply, on explicit `catalog publish`, and on release-tag events. Never regenerated mid-apply — consumers always see a snapshot tied to a specific `release_sha`.

**2. Resolver API** — three transport flavours.

| Method | Purpose |
|---|---|
| `resolve(urn)` | Lookup by URN |
| `list_contracts(filter)` | List by domain / classification / state |
| `lineage_in(column_urn)` | Resolve upstream column references |
| `lineage_out(column_urn)` | Resolve downstream consumers |
| `health()` | Returns `OK` / `STALE` / `UNAVAILABLE` with cache age |

Transports: **in-process** (same project, default for CLI), **HTTP REST** (cross-project, ETag-based caching, vault-resolved bearer auth), and **filesystem-static** (committed exports for air-gapped or GitOps deployments).

**3. Cache invalidation.** HTTP transport uses ETag matching `release_sha`, `Cache-Control: public, max-age=300` default, `X-Federation-Schema-Version` header. Clients respect 304 Not Modified. Filesystem transport invalidates per `git pull`.

**4. Availability and SLO.** 99.5% per-publisher export availability, p99 resolve latency <500 ms HTTP / <5 ms in-process, 24h stale-but-served grace window with `STALE` status.

**5. Degraded mode** — explicit and operation-dependent. Read-only browse serves stale with markers. **Fail-closed** for lint, apply against cross-project URNs, and AI-policy gates. Partial-graph display for lineage. The principle: anything gating security or correctness fails closed; anything gating convenience serves stale with explicit markers.

**6. Cross-project release semantics.** Producer-controlled default — consumers see new releases on next cache revalidation. Opt-in consumer-declared hard dependency (`block_on: [classification_increase, column_drop, deprecation]`) lets a consumer veto specific upstream changes. Bidirectional atomic release stays deferred.

## Consequences

- **Positive:** The largest single architecture risk closes — federation has an explicit schema, transport, cache, SLO, degraded mode, and release semantics that can be audited. Three transport flavours match real-world deployments without forcing customers into the wrong one. Degraded-mode behaviour is documented per operation, not implicit.
- **Negative / costs:** Substantial implementation surface (schema, HTTP endpoint, ETag handling, degraded-mode wiring, hard-dependency parsing). Schema version becomes a constraint with deprecation cycles. The 24h stale grace creates a correctness-versus-availability tension that auditors must accept. Hard-dependency declaration is opt-in, so most consumers will rely on lint to catch breaking changes.
- **Neutral:** Push-invalidation via event channel is reserved for the managed offering and ships separately.

## Related

- See [arch-05-catalog-federation](../architecture/arch-05-catalog-federation.md)
- Related ADRs: 009 (catalog), 020 (hub-and-spoke), 023 (lineage), 028 (resilience), 031 (project isolation), 032 (CI/CD)
