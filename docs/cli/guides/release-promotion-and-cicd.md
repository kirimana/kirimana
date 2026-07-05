# Release promotion & CI/CD

**Audience:** platform architects and release managers. **Read this** to understand how
Kirimana promotes a project forward through environments — the forward-only model, the
release manifest, the `kiri release` lifecycle, human approvals, and the reconcile sign-off
gate that fences a promotion behind validated data.

## Forward-only promotion

Kirimana promotes environments **forward only**: a git SHA that is live in `dev` is
promoted to `test`, then to `prod`. There is a single Kirimana, a single `main`, and a
**release manifest** that records which commit SHA is live in each declared environment.
Promotion is *applying a known-good SHA to the next environment* — never a divergent
per-environment branch. This keeps every environment traceable to one commit and makes
"what is live in prod, and where did it come from" a single lookup.

Environments are declared in `kiri.yml`; the manifest is seeded from them:

```bash
# Populate the release-state store from kiri.yml's environment config
kiri release init
```

## The release lifecycle

```bash
# What SHA is live in every environment?
kiri release status

# Diff HEAD (or --from) against the commit currently live in --to
kiri release plan --to test
kiri release plan --to prod --from v1.4.0        # any git ref
kiri release plan --to prod --domain sales        # scope to one domain

# Promote a SHA to an environment and stamp release_sha/tag/ts on success
kiri release apply --to test
kiri release apply --to prod --tag v1.4.0

# Which contract version is live in which environment?
kiri release matrix
kiri release matrix -o json                       # for CI / bots

# Recent promotions, newest first
kiri release history --env prod --limit 20
```

`kiri release apply` does **not** checkout — it promotes the SHA currently checked out
(defaulting to `HEAD`); if you pass `--sha` it must resolve to the same commit as `HEAD`.
The caller (CI job or a human) is responsible for having the right ref checked out. On
success it stamps `release_sha` (and optional `--tag`) into the manifest. `--domain` scopes
a promotion to one information domain; `--skip-dbt` / `--skip-fetch` are the usual build
escape hatches.

Use `--allow-dirty` only as a dev-only escape — a clean, committed tree is the norm for any
promotion that touches `test` or `prod`.

## Approvals

High-trust operations flow through an approval ledger rather than ad-hoc sign-off. Operators
vote on pending requests; **one reject vetoes**:

```bash
kiri approve list                 # pending requests for a workspace
kiri approve show <request-id>    # vote ledger + state
kiri approve grant <request-id>   # record an APPROVE vote
kiri approve reject <request-id>  # a single veto blocks the request
kiri approve issue <request-id>   # issue a use-once bearer token for an APPROVED request
```

Point `--store` at the same backend (`memory` / `jsonl` / `postgres`) that your AI surfaces
write to, or the operator view won't see the requests.

## The reconcile sign-off gate

For a promotion that follows a data migration, "the code is approved" is not enough — the
*data* must be validated. `kiri reconcile` runs post-migration data validation (source vs
target row counts, PK uniqueness, null rates, date bounds, SCD2 checks) and produces a
report:

```bash
kiri reconcile \
  --source-table sales_src.dbo.orders \
  --target-table prod_sales.silver.orders \
  --primary-key order_id \
  --columns customer_id,amount \
  --date-columns order_date \
  --format markdown -o reconciliation/orders.md
```

The reconcile result binds into the approval flow as a **sign-off gate**: an approval token
can be issued and consumed against the reconcile tuple, so a promotion is fenced behind a
green data-validation, not just a code review:

```bash
kiri approve reconcile-hash        # print the binding hash for a reconcile sign-off
kiri approve consume-reconcile     # consume a token bound to that reconcile tuple
```

Use `--report-mode locked` when the report must omit raw values for classification reasons.

## A typical CI flow

1. Merge to `main`; CI runs `kiri contract validate` + `kiri layer validate`.
2. `kiri release apply --to dev` on merge.
3. `kiri release plan --to test` surfaces the diff for review; promote with
   `kiri release apply --to test`.
4. For a migration, `kiri reconcile` must be green and signed off via `kiri approve
   consume-reconcile` before `kiri release apply --to prod --tag vX.Y.Z`.
5. `kiri release matrix -o json` feeds the deployment dashboard.

## Related reference

- [Releases & history](../releases-history.md) — the generated `kiri release` / `kiri approve` reference
- [The medallion + contracts model](./medallion-and-contracts-model.md) — contract states that promotion honours
- [Governance & catalog](./governance-and-catalog.md)
