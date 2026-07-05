# The medallion + contracts model

**Audience:** platform architects. **Read this** to build the mental model that every
other Kirimana surface assumes — bronze / silver / gold expressed as *data contracts*,
the ODCS v3 wire format, the `kiri.*` extension namespaces, the source-vs-contract split,
and the medallion state machine that governs promotion.

## Why contracts, not pipelines

Kirimana is metadata-driven. The unit of truth is not a dbt model or a notebook — it is a
**data contract**: one document that answers every operational, governance, and
AI-consumption question about a dataset (its source, endpoint, columns, transformation,
orchestration, governance, quality, and AI policy). Adapters, generators, the catalog, and
Kiri (the AI assistant) all read the *canonical model* parsed from that document — never a
hand-written pipeline. Platform artefacts (dbt SQL, DLT notebooks, native jobs) are
*compiled* from contracts; they are outputs, never the source.

The contract is the same document on every runtime. Only the compilation step is
platform-specific.

## The wire format: ODCS v3 + `kiri.*`

Contracts are [Open Data Contract Standard](https://bitol.io) v3 YAML, adopted straight —
no forks, no parallel fields. ODCS-native fields carry the structure Kirimana needs
directly: `name`, `version`, `status`, `domain`, `owner`, `schema[].properties[]`,
`quality[]`, `sla[]`, `servers[]`, `authentication[]`.

Where a need falls outside ODCS, Kirimana expresses it through `customProperties` under the
documented **`kiri.*`** namespace — for example `kiri.classification`,
`kiri.medallion.state`, `kiri.dv.*` (Data Vault), `kiri.gold.*` (Kimball star),
`kiri.lineage.*`, `kiri.schedule.*`, `kiri.ai_policy.*`. Each namespace is promoted to a
typed attribute on the canonical model; unlisted custom properties round-trip unchanged in a
pass-through `extras` bag. The authoritative list of what goes where lives in
**`docs/contracts/metadata-master.md`**.

Two rules matter to an architect:

- **Owner + `kiri.classification` are mandatory** on every contract, and on every column
  whose sensitivity differs from the contract level. The canonical-model validator fails
  closed if they are missing.
- **Secrets are references, never literals.** Connection blocks use `${vault:...}`; CI
  rejects detected plaintext.

Inspect any contract as the canonical model the AI actually sees:

```bash
kiri contract show contracts/sales/silver/customer.yml
kiri contract validate contracts/sales/silver/customer.yml   # structural + semantic
```

Emit the JSON Schema for ODCS v3 + the `kiri.*` extensions (useful for editor validation):

```bash
kiri contract schema-export -o kiri-odcs.schema.json
```

## The three layers, expressed as contracts

The medallion architecture is a property, not a directory convention. Each contract
declares which layer it belongs to via `kiri.medallion.state`:

| Layer | What the contract describes | Typical `kiri.*` extensions |
|---|---|---|
| **bronze** | Raw landed data with `_kiri_*` provenance columns, one contract per ingested object | `kiri.ingest.*`, `kiri.naming.bronze_ref_verbatim` |
| **silver** | Conformed, historized, business-ready entities (flat / Data Vault / Kimball technique — chosen project-wide) | `kiri.dv.*`, `kiri.dedup.*`, `kiri.historize` |
| **gold** | Serving layer — Kimball star schema facts and dimensions | `kiri.gold.*` (grain, measures, dimension refs, SCD type) |

Bronze contracts are typically *derived* from a source; silver and gold are *authored*
(with Kiri drafting the first pass). See the silver and gold guides for the modelling
detail; this page is about the shared shape.

## Source vs contract — a deliberate split

A **source** (`sources/*.yml`) is the *system of record* — a connection, an endpoint, a
cadence, credentials. A **contract** is the *governed dataset* produced from it. The
relationship is **1:N**: one source can feed many contracts.

The split is intentional. Attributes that describe *how data is delivered* live on the
source; attributes that describe *what the dataset guarantees* — classification, quality
rules, medallion state, load strategy, SLA — live on the contract. This is why you can
re-point a source (new host, rotated credential) without touching a single governance
guarantee, and why one CRM export can become both a bronze landing contract and a curated
silver entity with different owners.

```bash
kiri source validate sources/crm.yml     # source-YAML lifecycle
kiri contract scaffold --help            # render a bronze contract from a source URN
```

## The medallion state machine

Contracts move through a governed lifecycle, not free-form editing. The states are
`draft → bronze → silver → gold`, with `deprecated` as a terminal off-ramp. Promotion is an
explicit, audited transition — it is *not* implied by editing a file:

```bash
# Promote a contract one step along the state machine
kiri contract promote customer --to silver

# Pause runtime execution without demoting (flips kiri.medallion.active)
kiri contract promote customer --to silver --inactive
```

`--active` / `--inactive` decouple *lifecycle state* from *runtime execution*: an
architect can freeze a table's builds during an incident without rewriting its governance
state. Downstream compilation, orchestration, and lineage all honour these states —
`paused` / `deprecated` / `draft` contracts drop out of compiled workflows with a
machine-readable reason.

## How the model composes

```
source (sources/*.yml)          system of record, 1:N
   └─▶ bronze contract          raw + _kiri_* provenance
         └─▶ silver contract    conformed / historized (flat | DV | kimball)
               └─▶ gold contract  Kimball star (facts + dimensions)
```

Every arrow is declared lineage (`kiri.lineage.*`), which is what lets Kirimana walk from a
reporting goal all the way back to a source column — see
[column & goal lineage](./lineage-column-and-goal.md).

## Related reference

- Concept deep-dive: **`docs/contracts/metadata-master.md`**
  and **`docs/architecture.md`**
- [Contracts & schema](../contracts-schema.md) — the generated `kiri contract` command reference
- [Naming, catalog & schema registry](./naming-catalog-and-schema-registry.md)
- [Silver & Data Vault](../silver-datavault.md)
- [Gold & analytics](../gold-analytics.md)
