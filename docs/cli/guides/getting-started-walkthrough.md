# Getting started: an end-to-end walkthrough

This guide takes you from an empty machine to a materialised medallion warehouse on DuckDB — the local platform adapter — using nothing but the `kiri` CLI. Every command here runs against your own local files; no cloud account is needed.

By the end you will have: a project, one source, one validated contract, a plan, and an applied build with bronze and silver tables.

## 1. Install

Two ways to get `kiri`.

**From the source checkout** (for contributors, or to run the demo):

```bash
make dev      # install the uv workspace, including the kiri CLI
```

**From PyPI** (to use the CLI in your own data-warehouse project):

```bash
pip install kiri-cli
kiri --version
```

## 2. Initialise a project

`kiri init` bootstraps a new data-warehouse project as its own git repository. Three decisions matter at init time:

- **`--silver-technique`** (required, and locked after the first silver contract): `flat`, `data_vault`, or `kimball_dimensional`. See [silver ways of working](silver-layer-ways-of-working.md) if you are unsure — `flat` is the simplest starting point.
- **`--mode`** — how contracts are owned:
  - `contract` (default): you own a visible `contracts/` directory.
  - `auto`: Kiri owns hidden `.kiri/contracts/`; you graduate them later with `kiri graduate`.
  - `hybrid`: both.
- **`--layout`** — `per_domain` (default) or `per_source`.

```bash
kiri init sales_dw \
  --silver-technique flat \
  --mode contract \
  --layout per_domain \
  --adapter localduckdb
```

This scaffolds `sales_dw/` with a `kiri.yml`, the medallion folder structure, and a git repo (pass `--no-git` to skip). It refuses to scaffold inside the Kirimana source checkout — your project must be its own repository.

Make it the active project for this shell:

```bash
cd sales_dw
kiri use .
kiri project status     # confirms the active project + target
```

## 3. Add one source

A *source* is a declaration of where data comes from. For a first local run, the `fixture_file` mode reads CSV/JSON/XML from the project's `data/raw` directory — ideal for a demo.

Scaffold a source deterministically:

```bash
kiri source scaffold customers \
  --owner data-team@example.com \
  --domain sales \
  --classification internal \
  --mode fixture_file \
  --tables customers
```

Validate it and commit it atomically:

```bash
kiri source validate --content sources/customers.yml
kiri source commit --target sources/customers.yml --content sources/customers.yml
```

Drop a matching `customers.csv` into `data/raw/` so there is something to ingest. (For real sources — REST, database, Airbyte — see [adding a source](adding-a-source.md).)

## 4. Author and validate a contract

A *contract* is the ODCS v3 metadata master for one dataset — owner, classification, schema, and the `kiri.*` extensions that drive generation. Kiri can draft one for you:

```bash
kiri contract new \
  --source customers \
  --table customers \
  --name customer \
  --owner data-team@example.com \
  --classification internal \
  --domain sales
```

Kiri drafts the contract and you accept it. Every contract needs `owner` and `kiri.classification` — this is enforced. Validate before you build:

```bash
kiri contract validate contracts/sales/customer.yml --strict
```

`--strict` fails on semantic warnings too, not just hard errors. Fix anything it reports now — cheaper than debugging a failed apply later.

## 5. Plan

`kiri plan` shows what an apply *would* do and surfaces metadata conflicts, without touching any tables:

```bash
kiri plan --target dev
```

Read the per-source-table predicted changes and the silver/gold contract previews. For CI, add `--strict` to exit non-zero on any conflict, or `--format json` for a machine-readable summary you can diff between two plans.

## 6. Apply

`kiri apply` runs the full pipeline: ingestion → AI description → contracts → dbt build.

```bash
kiri apply --target dev
```

On DuckDB this materialises bronze from your landed files and builds the silver models for your technique. Scope a re-run to one contract or domain while iterating:

```bash
kiri apply --target dev --contract customer
kiri apply --target dev --domain sales
```

## 7. Confirm the result

```bash
kiri list-generated          # every file the generator owns
kiri silver plan             # the locked technique + planned silver contracts
kiri silver conformance      # technique-appropriate conformance + evidence
```

You now have an end-to-end medallion build on DuckDB. From here:

- Add more sources — [adding a source](adding-a-source.md)
- Understand the build lifecycle and debugging — [plan, apply & debugging](plan-apply-and-debugging.md)
- Choose your silver modelling approach deliberately — [silver ways of working](silver-layer-ways-of-working.md)
- Point at a real target and wire secrets — [targets, profiles & secrets](targets-profiles-and-secrets.md)

## Related reference

- [getting started](../getting-started.md) — `kiri init`, `kiri project`, `kiri use`, `kiri targets`
- [sources & ingestion](../sources-ingestion.md) — `kiri source`, `kiri ingest`
- [contracts & schema](../contracts-schema.md) — `kiri contract`
- [build, plan & apply](../build-plan-apply.md) — `kiri plan`, `kiri apply`, `kiri list-generated`
