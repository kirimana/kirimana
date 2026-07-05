# Silver-layer ways of working

The silver layer is where raw ingested data becomes trustworthy, modelled entities. Kirimana gives you two independent choices:

1. **The project-wide *technique*** — locked once, at `kiri init`, for the whole project (`flat`, `data_vault`, or `kimball_dimensional`). This is a strategic decision about *how you model history and relationships everywhere*.
2. **Per-contract *shape and zone*** — set with `customProperties` on individual contracts (`kiri.silver.kind`, `kiri.medallion.silver_zone`). This tells Kirimana what a given table *is* within the chosen technique.

Everything below tells you which knob to turn for a given outcome, and the exact CLI path to plan, apply, and verify it. The commands (`kiri silver plan`, `kiri silver apply`, `kiri silver conformance`, `kiri dv apply`) are **technique-agnostic**: the same command works whichever technique you locked. What changes is the contract metadata it reads.

## Decision tree

```
Start: what does your silver layer need to be?
│
├─ Pass-through / projection-complete tables you author yourself
│     (rename, cast, cleanse — no generated history)
│        → technique: flat
│        → build with: kiri silver apply   (delegates to kiri apply)
│
├─ Full auditable history as hubs / links / satellites
│     (raw vault + business vault, insert-only, source-effective)
│        → technique: data_vault
│        → build with: kiri dv apply
│        → optional engine: kiri dv install  (DataVault4dbt backend)
│
├─ Star-schema-style SCD2 dimensions (type_1 / type_2 / type_3 / type_7)
│        → technique: kimball_dimensional
│        → build with: kiri silver apply
│
└─ A generated "standardized → curated SCD2" shape off a relational source
      (type-cast standardized, then generated SCD2 history)
         → technique: any of the above, PLUS per-contract
           kiri.silver.kind: scd2_curated
         → build with: kiri silver apply
```

The technique is a whole-project switch. `scd2_curated` is a *per-contract shape* layered on top — you can have `scd2_curated` contracts alongside plain ones in the same project.

## The three project-wide techniques

You choose the technique at project creation and it is locked once the first silver contract exists:

```bash
kiri init sales_dw --silver-technique flat
kiri init sales_dw --silver-technique data_vault
kiri init sales_dw --silver-technique kimball_dimensional
```

| Technique | What silver looks like | How you build it | Notes |
|---|---|---|---|
| `flat` | Pass-through / projection-complete models you author in SQL. Rename, cast, cleanse; no generated history. | `kiri silver apply` (delegates to `kiri apply`) | Simplest. Conformance checks projection-preservation (nothing silently dropped). |
| `data_vault` | Hubs, links, satellites. Raw vault = full SCD2 history; business vault = merged current-state entities. Insert-only, source-effective sequencing. | `kiri dv apply` | Optional `DataVault4dbt` engine via `kiri dv install`; the native backend needs no install. |
| `kimball_dimensional` | SCD2 dimensions (`type_1` / `type_2` / `type_3` / `type_7`) with explicit business keys and surrogate strategy. | `kiri silver apply` | Dimensions declared with `kiri.silver.scd_type`, `kiri.silver.business_keys`. |

Preview which technique is locked and which contracts will be planned:

```bash
kiri silver plan
```

`kiri silver plan` is read-only and produces the same output shape for every technique.

## The generated `scd2_curated` shape

A relational-warehouse rebuild usually targets a **standardized → curated** silver: a type-cast/dedup *standardized* current-state, then a *curated* layer that builds SCD2 history. Kirimana can **generate** that shape from contract metadata rather than having you hand-author the SCD2 SQL per entity.

You opt a contract into it with a `customProperty`:

```yaml
customProperties:
  - property: kiri.silver.kind
    value: scd2_curated
  - property: kiri.silver.business_keys
    value: [customer_id]
  - property: kiri.silver.effective_from_column
    value: valid_from        # a BUSINESS-effective timestamp
```

### `effective_from_column` must be a business-effective timestamp

The SCD2 sequencing key is a **declared source business-effective timestamp** — the moment the fact became true in the source system. It is **never** the ingest or load timestamp.

This is enforced, not advisory. If the sequencing key resolves to an ingest/load timestamp when a backfill is in scope, validation **fails at parse time**. The reason: bulk-loading full source history and sequencing by ingest time collapses every version of a row into a single current row — silent history loss. Kirimana turns that trap into an error rather than letting it corrupt your history.

```bash
# Validate before you build — the effective_from rule is checked here.
kiri contract validate contracts/sales/customer.yml --strict
```

### `scd2_curated` and the raw / business zones

Under a `scd2_curated` contract, `kiri.medallion.silver_zone` selects between two sub-layers:

- **`raw` (default)** — the **generated** standardized current-state plus the curated SCD2 satellites. Requires `business_keys` + `effective_from_column`.
- **`business`** — a **latest-only, hand-authored** business entity (for example a model that joins two satellites and applies survivorship rules). It is silver→silver lineage, carries no SCD2 columns of its own, and is therefore **exempt** from the `scd_type` / `effective_from_column` / `business_keys` requirements — but declared business-rule invariants still apply. The generator leaves it alone.

```yaml
# A hand-authored business-vault entity under scd2_curated
customProperties:
  - property: kiri.silver.kind
    value: scd2_curated
  - property: kiri.medallion.silver_zone
    value: business
```

## Zones via `kiri.medallion.silver_zone`

`kiri.medallion.silver_zone` marks *where in the silver layer* a contract sits, and each zone has a **fail-closed read barrier** — a contract in one zone may only reference contracts in the zones it is allowed to read. A reference outside the allowed set is a validation error, not a warning.

### Legacy zones

| Zone | Meaning | May read |
|---|---|---|
| `raw` | Full-history sub-layer (satellites under Data Vault / `scd2_curated`). | — (must not depend on `business`) |
| `business` | Opt-in merged current-state entities with business rules. | Its upstream silver, per technique |
| `pit` | Point-in-time / bridge tables. A leaf zone. | Its declared temporal upstreams |

`business` silver is opt-in — a project with no `business`-zone contracts is unaffected; the default is `raw`.

### Governed named peers

These are first-class zones with the same fail-closed posture, for teams that model a standardized → curated → conformed information architecture:

| Zone | Meaning | May read (the read barrier) |
|---|---|---|
| `standard` | SCD2 standardized entities directly off bronze. | `bronze` only — no silver table |
| `curated` | Facts / business-logic tables. | silver `standard`, `conformed`, and same-zone `curated` — **not** `raw`, `business`, `pit`, or bronze |
| `conformed` | Master / information-model (conformed) entities. | silver `standard` and cross-domain `conformed` — **not** `curated` |

Set the zone on the contract:

```yaml
customProperties:
  - property: kiri.medallion.silver_zone
    value: curated
```

The barriers are enforced wherever one silver contract references another — Data Vault `kiri.dv.connects` / `kiri.dv.attached_to`, and layer-manifest upstream declarations. If you need a purely descriptive label that carries no barrier, use a governance label instead of overloading a zone.

## For each combination: what sets it, and how to build it

| Outcome | Technique (at `kiri init`) | Contract `customProperties` | Build command |
|---|---|---|---|
| Pass-through / projection silver | `flat` | (none required) | `kiri silver apply` |
| Hubs / links / satellites, full history | `data_vault` | `kiri.dv.*`, `kiri.semantic.business_keys` | `kiri dv apply` |
| Data Vault with DataVault4dbt engine | `data_vault` | as above | `kiri dv install` once, then `kiri dv apply` |
| SCD2 dimensions | `kimball_dimensional` | `kiri.silver.kind: dimension`, `kiri.silver.scd_type`, `kiri.silver.business_keys` | `kiri silver apply` |
| Generated standardized → curated SCD2 | any | `kiri.silver.kind: scd2_curated`, `business_keys`, `effective_from_column` | `kiri silver apply` |
| Generated SATs (raw sub-layer) | any + `scd2_curated` | `kiri.medallion.silver_zone: raw` (default) | `kiri silver apply` |
| Hand-authored latest-only business entity | any + `scd2_curated` | `kiri.medallion.silver_zone: business` | `kiri silver apply` (you author the SQL) |
| Standardized-off-bronze zone | any | `kiri.medallion.silver_zone: standard` | `kiri silver apply` |
| Facts / business-logic zone | any | `kiri.medallion.silver_zone: curated` | `kiri silver apply` |
| Master / conformed model zone | any | `kiri.medallion.silver_zone: conformed` | `kiri silver apply` |

Verify any of them:

```bash
# Technique-appropriate conformance + compliance evidence
kiri silver conformance

# For flat / Kimball / scd2_curated: also run the live per-run quality report
kiri silver conformance --report --target dev --report-format md

# Data Vault: cross-backend (native vs DataVault4dbt) conformance
kiri dv conformance
kiri dv quality run
```

## When to use which

- **`flat`** — you want full control of silver SQL, your sources are already reasonably modelled, and history is handled elsewhere or not needed. Lowest ceremony.
- **`data_vault`** — you need fully auditable, insert-only history with clean separation of raw ingestion from business logic, and you expect many overlapping sources. Highest governance, most structure.
- **`kimball_dimensional`** — your primary consumer is BI and you want conformed SCD2 dimensions feeding star schemas.
- **`scd2_curated`** (per-contract) — you are rebuilding a relational warehouse and want the standardized → curated SCD2 shape *generated* with correct source-effective sequencing, without hand-authoring SCD2 SQL per entity. Use `silver_zone: raw` for the generated SATs and `silver_zone: business` for the few hand-authored merge entities.
- **Named zones (`standard` / `curated` / `conformed`)** — you want the read barriers to enforce a layered information architecture at PR time, so a `curated` fact can never accidentally read straight from bronze.

## Related reference

- [silver & data vault](../silver-datavault.md) — every `kiri silver` and `kiri dv` command
- [contracts & schema](../contracts-schema.md) — `kiri contract validate`, `show`, `promote`
- [build, plan & apply](../build-plan-apply.md) — `kiri plan`, `kiri apply`, `kiri verify`
- [gold & analytics](../gold-analytics.md) — star schemas that consume silver
