# arch-05: Catalog and Federation

Kirimana's catalog is a derived view: it is generated from contracts, not authored separately. Where multiple Kirimana projects coexist — different domains, different business units, different teams — they federate through a single resolver protocol so a downstream consumer can bind to an upstream contract that lives in another repository. This document describes the hub-and-spoke governance model, the federation resolver, the pluggable catalog backend, and how Kirimana integrates with existing enterprise catalogs.

## Hub-and-spoke governance

A Kirimana project is a governance boundary: one project, one catalog, one main branch, three environment cursors (arch-08). The hub-and-spoke model lifts that boundary one level: a central platform team owns shared policy (RBAC roles, compliance controls, naming conventions) and each domain team owns its own project under that policy. Domains run autonomously inside the guardrails the hub publishes.

Concretely:

- **Hub** publishes the policy module: RBAC, governance mode (light vs enterprise), shared naming conventions, compliance pack selection.
- **Spokes** are individual Kirimana projects, each scoped to a domain. Each has its own contracts, its own apply cadence, its own approvals.
- **Cross-domain** dependencies flow through the federation resolver (below), not by sharing repositories or catalogs.

Domain is a first-class concept in the CLI (`--domain`), in customProperties (`dca.domain.*`), and in audit (every event carries the domain). RBAC roles are scoped by environment and domain.

## Federation resolver

When a consumer contract in project B binds to a source contract in project A, the binding is a URN: `urn:kirimana:contract:<domain>.<dataset>:<version>`. The federation resolver takes that URN and returns the source contract.

The resolver is a typed protocol with three transports:

- **in-process** — both projects loaded in the same Python process (development, monorepo deployments)
- **HTTP REST** — each project publishes a federation endpoint; resolver fetches over the wire
- **filesystem** — each project writes a versioned public JSON snapshot to a shared location (object storage, mounted volume); resolver reads from there

Federation publishes only the public face of a project — contract identifiers, schemas, lineage URNs, deprecation state. It does not publish data, credentials, or internal state. The SLO is 99.5% availability with p99 latency under 500ms for HTTP. Resolution is cached with a TTL; staleness is surfaced to the caller through a typed health report (arch-08).

Two failure modes are distinguished:

- **Browse path** — listing contracts for the catalog UI. Stale-with-marker is acceptable.
- **Security/release path** — checking whether a downstream may apply when its upstream is deprecated. Fail-CLOSED. A federation timeout blocks the apply rather than silently passing.

Cross-project release blocking is opt-in per consumer. A consumer that declares a hard dependency on an upstream cannot promote past a deprecation; a soft dependency only warns.

## Pluggable catalog backend

The catalog backend is itself an adapter (`CatalogBackend` protocol, arch-03). The default is SQLite, suitable for single-node deployments and CI. Postgres ships as the production backend for the web app and managed deployments. Larger installations can implement their own backend against the protocol.

The catalog stores asset records, column-level lineage edges, review-state for attributes (`new` → `in_progress` → `submitted` → `approved` | `rejected`), and quality-evidence pointers. It does not store the contract itself — the contract YAML in the repository is canonical, the catalog is a query-optimised projection.

## Integration with Unity Catalog, Horizon, Purview

For customers running an enterprise catalog already, Kirimana federates rather than replaces. The `CatalogSourceAdapter` protocol lets Kirimana pull metadata from Unity Catalog, Snowflake Horizon, or Microsoft Purview into its own catalog as read-only entries. Overlapping assets resolve source-wins: the external catalog is authoritative for what exists; the Kirimana contract is authoritative for what should exist. The plan/apply/verify cycle (arch-02) reconciles the two.

V1 adapters for Unity Catalog, Horizon, and Purview are present as proposed/beta. They are unidirectional today — read from the external catalog into Kirimana. Push-back (Kirimana writes governance metadata into Unity Catalog) is on the roadmap but not in v0.9.

## Related ADRs

011, 020, 021, 022
