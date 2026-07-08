# Authoring Kirimana YAML — overview

A Kirimana project is configured through **YAML you edit by hand** (or with Kiri's
help). This guide set is the operator's manual for those files: what each block
means, what values are legal, and how to make common changes safely.

There are four files an operator works in:

| File | What it configures |
|---|---|
| `kiri.yml` | The **project**: name, adapter, targets/connections, silver technique, naming, secrets, governance. One per project, at the repo root. |
| `01-config/<domain>/bronze.yml` | The **bronze layer** for a domain: raw tables landed from sources. |
| `01-config/<domain>/silver.yml` | The **silver layer**: cleaned/standardized (and, per technique, historised or vaulted) tables. |
| `01-config/<domain>/gold.yml` | The **gold layer**: analytics-facing dimensions and facts. |

The three `01-config/<domain>/*.yml` files are **layer manifests** — one ODCS v3
`DataContract` document per layer, listing that layer's tables. `kiri.yml` is the
project config that binds them to a runtime.

## The worked example: `sales`

Every guide in this set uses one generic example — a `sales` domain with a simple
order-to-cash lineage. It ships in the repo at `examples/sales/` and validates
offline against a local DuckDB target, so you can copy any snippet and try it:

```
bronze.sales_order ─┐
                    ├─► silver_sales_order ─► fct_sales_order   (gold fact)
bronze.customer ────┴─► silver_customer ───► dim_customer       (gold dimension)
```

- **bronze** — two raw feeds: `sales_order` and `customer` (the latter carries PII).
- **silver** — flat technique: one standardized table per bronze table (1:1 rename + cast + dedup).
- **gold** — `dim_customer` (a Type-1 dimension) and `fct_sales_order` (an order-grain fact).

Run it yourself:

```bash
uv run kiri project validate --project examples/sales   # kiri.yml is well-formed
uv run kiri layer validate   --project examples/sales   # the 3 manifests are coherent
uv run kiri doctor           --project examples/sales   # full health punch-list (clean)
```

## How the pieces relate (ODCS v3 + `kiri.*`)

The layer manifests are **ODCS v3 data contracts** (the open Open Data Contract
Standard — Kirimana's canonical format, per ADR 0002). Everything Kirimana adds on
top lives under documented `customProperties` in the `kiri.*` namespace:
`kiri.layer`, `kiri.classification`, `kiri.gold.kind`, `kiri.lineage.from_columns`,
and so on. ODCS carries the *shape* (tables, columns, types); `kiri.*` carries the
*platform intent* (which layer, how to materialise, governance, lineage).

The authoritative field-by-field spec for the per-column/per-table `kiri.*`
properties is the contract metadata master, published at
`https://github.com/kirimana/kirimana/blob/main/docs/contracts/metadata-master.md`.
This guide set is the operator-facing companion: it shows **where** each block goes
in the four files and **how** to edit it, and links out to that spec for the deep
semantics of individual properties.

## Four rules that apply everywhere

1. **Precedence: column > table > layer > default.** A `customProperty` set at the
   layer level (e.g. `kiri.classification: internal` on `bronze.yml`) applies to
   every table and column below it, unless a table or column overrides it. Set the
   common case once at the layer; override the exceptions.

2. **Enums are fail-closed.** Where a field takes a fixed set of values (silver
   `technique`, `kiri.gold.kind`, `auth_type`, …), an unknown value is rejected at
   parse time with a "did you mean" hint — it is never silently ignored. If a value
   doesn't validate, it isn't in the allowed set.

3. **Secrets are references, never literals.** Connection secrets use
   `${vault:<id>:<key>}` and resolve through the configured `vault` backend
   (ADR 0011). CI fails on detected plaintext secrets. Never paste a token into YAML.

4. **Names are preserved verbatim when you ask.** With `naming.preserve_verbatim:
   true` (ADR 0128) the physical object names equal the contract's declared names
   byte-for-byte — case, diacritics, and all. This matters for migrations that must
   match an existing warehouse exactly.

## How to use this guide set

- **Looking up a block or field?** Go to the reference:
  - [Editing `kiri.yml`](authoring-kiri-yml.md) — every project-config block.
  - [Editing layer manifests](authoring-layer-manifests.md) — bronze/silver/gold,
    at the layer / table / column levels.
- **Making a change?** Go to [Authoring tasks](authoring-tasks.md) — recipes like
  "add a source", "add a silver table", "switch silver technique", "set up a
  localized twin", each as *goal → minimal diff → verify*.
- **Something failed?** See [Validating your YAML](authoring-validation.md) — the
  commands above plus a "error → cause → fix" table.

For the concepts behind the choices (why a contract, medallion, silver techniques,
naming/catalog), see the sibling guides
[Medallion and the contract model](medallion-and-contracts-model.md),
[Silver-layer ways of working](silver-layer-ways-of-working.md),
[Naming, catalog and the schema registry](naming-catalog-and-schema-registry.md),
and [Targets, profiles and secrets](targets-profiles-and-secrets.md).
