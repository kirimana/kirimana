# Naming, catalog & the schema registry

**Audience:** platform architects and data-platform engineers. **Read this** to control the
*customer-facing* physical names Kirimana emits — catalogs, per-layer schemas, named silver
zones, and the URN scheme — and to understand the naming-lock that freezes those names once
data has been written.

## The problem: your names, not ours

A metadata-driven tool must not stamp its own vocabulary onto a customer's warehouse. Every
physical name Kirimana emits — catalog, schema, table — is configurable so the generated
platform looks like it was hand-built to the customer's conventions. Three mechanisms cover
it: the **catalog pattern**, the **per-layer / named-zone schemas**, and the **schema
registry** for arbitrary named schemas. All three are frozen by the **naming-lock** after
the first silver apply.

## Catalog pattern

`naming.catalog_pattern` in `kiri.yml` renders a catalog name from three tokens:

| Token | Resolves to |
|---|---|
| `{env}` | the target's environment (`dev` / `test` / `prod` / …) |
| `{domain}` | the contract's business domain (`sales`, `crm`, `hr`, …) |
| `{layer}` | the medallion state (`bronze` / `silver` / `gold`) |

For example a pattern like `{env}_{domain}` routes a `prod` `sales` contract into a
`prod_sales` catalog, while `{env}_lakehouse` collapses everything per environment. Explicit
per-target catalog overrides win over the pattern when you need an exception. Projects that
set no pattern are byte-unchanged — the feature is opt-in and additive.

## Per-layer medallion schemas + named silver zones

Each medallion layer resolves to a schema. Defaults follow the project convention
(`bronze_<project>`, `silver_<project>`, `gold_<project>`), overridable in the `naming:`
block of `kiri.yml`.

Silver additionally supports **governed named zones** beyond the legacy `raw` / `business` /
`pit`. The named vocabulary is `standard` / `curated` / `conformed`, each with its own
schema and a fail-closed read barrier:

| Zone | May read upstream from |
|---|---|
| `standard` | `bronze` only |
| `curated` | silver `standard`, `conformed`, `curated` |
| `conformed` | silver `standard`, `conformed` |

Defaults are `silver_<project>_standard`, `silver_<project>_curated`, and
`silver_<project>_conformed`. Zones are declared in the layer manifest and validated
*before* runtime, so a curated table that tries to read bronze directly fails validation, not
production.

## The schema registry (`kiri.target_schema`)

For anything the medallion defaults don't express, a layer manifest can declare a **registry
of named schemas** and route each table to one by label. A contract sets
`kiri.target_schema` to a registry label (`landing` / `standard` / `curated` / `serving` /
your own), and the resolver maps that label to an explicit physical schema name — escaping
the derived `<domain>_<schema>` slot entirely.

`kiri.target_schema` is a **closed-menu reference**: an unknown label is rejected against the
declared set (with a did-you-mean hint). This is the same registry that both `kiri compile
layer` and the dbt-generate path consume, so a table lands in exactly one schema regardless
of which build route produced it.

## URN scheme

Every governed asset carries a stable URN in the `kiri:` scheme, independent of physical
names:

```
kiri:contract:silver:customer
kiri:asset:silver:customer
```

URNs are what lineage, the catalog, and federation reference — so re-pointing a catalog
pattern or renaming a schema never breaks a lineage edge. Physical names are a rendering
concern; URNs are identity.

## Inspect and validate layer bindings

Work with the per-`(domain, layer)` manifests that hold the schema/catalog/zone bindings:

```bash
# List every discovered 01-config/<domain>/<layer>.yml
kiri layer list

# Render the resolved view of one (domain, layer) for a named target
kiri layer show sales silver --target prod
kiri layer show sales silver --target prod --format json

# Run the per-(domain, layer) validator (registry labels, zone barriers, coverage)
kiri layer validate

# Diff bindings between two targets
kiri layer diff --help
```

## The naming-lock

Naming is a **strategic, one-time decision**. The `naming:` block (patterns, per-layer
schemas, zone overrides) is **locked after the first silver write** — the same lock that
freezes the silver technique.

On that first write the manifest records a `naming_locked_at` timestamp and a canonical
snapshot of the naming block. On any subsequent apply the adapter reads the stored snapshot
in the same round-trip it already uses and refuses to run if the incoming `naming:` block
diverges, raising a `NamingLockError` that names the locked vs incoming snapshot. This is
what makes generated names *stable and safe to reference* downstream: a BI report bound to
`prod_sales.gold.dim_customer` cannot have the schema silently renamed under it.

Decide naming up front, validate it with `kiri layer validate`, then apply.

## Related reference

- [Governance & catalog access](../governance-catalog-access.md) — the generated `kiri layer` / `kiri catalog` reference
- [The medallion + contracts model](./medallion-and-contracts-model.md)
- [Gold star schema & localized twins](./gold-star-schema-and-localized-twins.md) — localized schema patterns build on this registry
