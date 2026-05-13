# ADR 023 — Column-level lineage

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** original ADR 0045 (archived)

## Context

Kirimana carries **contract-level lineage** through a small set of URN fields: `source_urn`, Data Vault `attached_to` and `connects`, Kimball `dimension_refs`. The DV Modeler renders these as a graph, and the goal-to-data USP traces a business goal to the contracts that feed it. This works at the contract grain — "this gold fact derives from these silver hubs".

It does NOT work at the **column grain**. Today, the only source of column lineage is the dbt manifest from the most recent apply. That has four problems:

1. **It is a runtime artefact, not contract metadata** — only exists after a successful apply, only on the machine that ran it, only while the manifest survives.
2. **It is dbt-coupled** — a future framework adapter (SQLMesh, raw SQL, Spark) might not produce a compatible manifest.
3. **AI cannot answer column-level lineage from ODCS alone** — Kiri must either run dbt (slow, requires a warehouse) or guess from column names (lossy).
4. **PR-time lint cannot enforce it** — there is no field to inspect, so reviewers spot missing provenance by reading SQL, exactly the human bottleneck shift-left exists to remove.

## Decision

Every property in an ODCS contract gains an optional `customProperties.dca.lineage.from_columns` field — a list of upstream column URNs:

```
urn:kirimana:<layer>:<contract_name>::<column_name>
urn:kirimana:source:<source_name>::<column_name>
```

The `::` separator keeps the URN unambiguous across layers.

**Required vs optional by layer:**

| Layer | Behaviour |
|---|---|
| bronze | `from_columns` may declare a single source-column URN; lint warns if absent. |
| silver | **Mandatory** for every property. Empty list = "originates here" (hash keys, sequence keys, derived flags). Missing field = lint error. |
| gold | **Mandatory** for every property; same rules. |

The empty-list-versus-missing-field distinction is load-bearing: empty list means deliberate, missing field means forgotten.

**Three parser invariants enforced at load:**

1. URN syntax matches `urn:kirimana:(source|bronze|silver|gold):*::*`.
2. Layer monotonicity — silver references source/bronze/silver only; gold references silver/gold only.
3. Self-reference is allowed within a contract.

Cross-contract resolvability is checked by lint at PR time, not by the parser, so partially authored contracts still load.

**Two new lint rules:**

- `column-lineage-missing` (warning, escalating to error after one minor release).
- `column-lineage-unresolvable` (error) — declared URN does not resolve to a known contract+column.

Contract-level lineage fields remain — they describe topology, while `from_columns` describes derivation. OpenLineage emission becomes a consumer of contract-declared lineage, not a parallel source of truth. Catalog `contract_urns` (which contracts a column appears in) is orthogonal and does not gain a `from_columns` mirror.

Alternatives rejected: keeping lineage in the dbt manifest only (not contract metadata, framework-coupled); storing in the catalog only (inverts the spec-driven model); using OpenLineage facets as the source (not readable at PR time, not surviving non-OL runtimes); auto-deriving from SQL parsing (famously hard for window functions, lateral joins, dynamic SQL — correctness budget gone).

## Consequences

- **Positive:** Audit-grade column lineage from contract YAML alone — no warehouse, no apply needed. AI traceability extends to the column grain. PR-time lint catches drift the moment a renamed upstream column hits CI. Framework-portable — switching transformation engines does not lose lineage. OpenLineage events carry a single declared source. DV Modeler gains column-level edges.
- **Negative / costs:** Authoring overhead — a 30-column contract gets 30 new entries (mitigated by an AI-assisted populator skill and the empty-list shortcut). Migration burden for the existing corpus (lint warns first, then errors). Two lineage layers (contract URNs and column URNs) describe the same world at different grains; parser cross-checks them but the redundancy is real. Stale lineage in long-untouched contracts can go unnoticed without a nightly catalog-wide lint job.
- **Neutral:** Transformation-expression recording (storing the SQL formula alongside the column list) is deliberately deferred — the string-list form is forward-compatible.

## Related

- See [arch-06-audit-lineage](../architecture/arch-06-audit-lineage.md)
- Related ADRs: 006 (ODCS), 021 (federation resolver), 025 (quality evidence)
