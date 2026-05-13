# ADR 007 — Contract lifecycle and state machine

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** originally ADR 0012 (archived)

## Context

ODCS (see ADR 006) is the canonical contract format, but the spec on its own does not say how Kirimana should run a contract through the medallion lifecycle, nor does it separate the physical binding to a data source from the consumer-facing agreement about a dataset view.

Early implementations of Kirimana conflated two different concerns into a single file: the physical binding (endpoint, auth, format) and the consumer-facing agreement (schema, classification, quality rules, freshness SLA). That shape breaks a relationship that is intrinsically one-to-many — a single physical source table can feed multiple analytical contracts, each with its own materialisation depth, cadence, and quality bar. Conflating them forces a single set of fields per source table, leaving the second contract no place to live.

A separate gap: contracts had no explicit lifecycle state. There was no way to express "this contract is authored but not yet running" versus "this contract drives the full pipeline through gold" versus "this contract is paused for the week without losing its position".

## Decision

Split contract metadata into two artefacts with explicit roles, link them by URN, and give every contract two orthogonal state fields.

### 1. Source / Contract split

- **Source** (`sources/*.yml`) — physical binding only: endpoint, auth, format, landing-zone mapping. Owned by the producing system. One row per `(source, table)`. Knows where and how to pull — nothing about when or for whom.
- **Contract** (`contracts/*.yml`, ODCS v3) — consumer-facing agreement: logical schema, classification and PII, data-quality rules, load rules, target medallion state, freshness SLA, consumer-side owner. One contract per dataset view.

The relationship is one-to-many (source to contracts). A contract points to exactly one source-and-table; a source can be referenced by any number of contracts.

### 2. URN binding

A contract references its source via a Kirimana URN string stored as an ODCS `customProperty`:

```yaml
customProperties:
  - property: dca.source_urn
    value: urn:kirimana:source:<source_name>:<table_name>
```

URN is the catalog's native reference shape (see ADR 006) and the lingua franca of external catalogs (DataHub, Atlan, Unity), so contracts plug into lineage and catalog touchpoints with zero translation.

### 3. Two orthogonal state fields

**Lifecycle state** — medallion depth, linear enum:

```
draft → bronze → silver → gold → deprecated
```

`apply` executes phases up to the contract's current state and stops. Promotion is an explicit edit, AI-assisted suggestion, or `kiri contract promote` command.

**Activation state** — boolean, `active` or `inactive`. Inactive means the runtime skips the contract entirely — a temporary pause that does not demote medallion depth. A gold contract can sit inactive for a week and come back as gold.

Both states live under `dca.lifecycle.*` customProperties. `deprecated` (lifecycle) and `inactive` (activation) both result in "do not run" but carry different intent: deprecated is permanent, inactive is paused.

### 4. Cadence reconciliation

Each contract declares its freshness SLA on itself. When multiple contracts bind to the same source URN, the runtime resolves the effective fetch frequency to the minimum interval across all active contracts. Contracts with longer cadences pick up only the cycles they care about. The source itself has no veto; rate-limit concerns surface as runtime warnings.

## Consequences

- **Positive:** The one-to-many relationship becomes expressible; AI-generation targets the contract, not a legacy combined file; catalog and lineage alignment is free because URN is the native reference shape; the state machine makes `apply` predictable; pause-without-demotion lets ops park a gold contract briefly without losing its position.
- **Negative / costs:** Two files per dataset where one existed — more ceremony for the trivial case (mitigated by `kiri contract add --source-urn` scaffolding); URN strings are less readable than structured `{name, table}` fields (accepted because contracts are AI-generated, not hand-edited); orthogonal state fields force the UI to teach two concepts where one compound enum would be simpler.
- **Neutral:** Activation state overlaps semantically with ODCS native `status`; the two stay separate until ODCS offers a pause semantic upstream.

## Related

- See [arch-02-contracts](../architecture/arch-02-contracts.md)
- Related ADRs: 006, 009, 010
