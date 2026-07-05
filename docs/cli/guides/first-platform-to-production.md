# From first startup to production: your first endpoint and medallion

This is the end-to-end runbook: from `pip install` to a **live source endpoint
and a full bronze → silver → gold medallion running on Databricks**, governed
and scheduled. It stitches the focused guides together in order and points at
the reference **skills** that let Kiri drive each step for you.

Each step below has a **do-it-yourself** command line and a **let-Kiri-drive-it**
skill. Install the skills once with `kiri skills install <name>` (or `--all`),
then invoke one with `/<skill-name>` in Claude Code — or ask your assistant to
run it via the [MCP setup](mcp-databricks-and-kiri-setup.md). Both paths do the
same thing — the skills just add the guardrails and prompts.

---

## 0. Prerequisites

- **Python 3.12+** and the CLI with the Databricks extra:
  ```bash
  pip install "kiri-cli[databricks]"
  kiri --version
  ```
- **A Databricks workspace** you can reach: host URL, a **PAT** (scoped to a
  service principal), and a **SQL warehouse id**.
- **The AI gateway** configured if you want Kiri's help —
  [AI gateway setup](ai-gateway-setup.md) (one `ANTHROPIC_API_KEY` is enough).
  Optional but recommended.
- **Optional:** wire [MCP](mcp-databricks-and-kiri-setup.md) so your assistant
  can drive the CLI and inspect the workspace in one conversation.

---

## 1. Scaffold the project — and decide the names now

```bash
kiri init sales_dw --silver-technique kimball_dimensional
cd sales_dw
kiri use .
kiri project status        # confirm the active project + target
```

Two decisions are **locked after the first silver apply** and are expensive to
change later (**ADR 0020**,
[silver technique](silver-layer-ways-of-working.md)):

- **Silver technique** — `flat`, `data_vault`, or `kimball_dimensional` (set at
  `init`, project-wide).
- **Customer-facing naming** — catalog pattern, medallion / silver-zone schema
  names, job prefix, governance schema. These live in `kiri.yml` and are **not**
  captured by `kiri init`. Set them before you build anything.

> **Let Kiri drive it:** **`databricks-project-setup`**
> walks you through sizing, scaffold, naming, and target wiring in one guided
> pass — it's the recommended way to start on Databricks.

---

## 2. Connect the Databricks target

Point a named target (e.g. `prod`) at the workspace and put every secret in the
vault — **never** in `kiri.yml` (**ADR 0011**):

```bash
kiri vault set databricks_token        # prompts; stored as a ${vault:…} ref
# host, http-path/warehouse, catalog paths → target config in kiri.yml
kiri project status                     # target resolves + connects
```

See [targets, profiles & secrets](targets-profiles-and-secrets.md) and
[Databricks & adapters](databricks-and-adapters.md) for the exact target block
and the Delta adapter options.

---

## 3. Add your first source endpoint

```bash
kiri ingest new-source --interactive    # AI-assisted; you pick the engine
# …or scaffold explicitly:
kiri source scaffold customers
kiri source validate --content sources/customers.yml
kiri source commit --target sources/customers.yml --content sources/customers.yml
```

Kirimana never auto-picks an ingest engine — you choose among `airbyte`,
`native`, `dlt`, `debezium`, or `landing_zone`
(**ADR 0028**).

> **Let Kiri drive it:** **`add-source`** —
> backend-agnostic source onboarding with the governance prompts built in.

---

## 4. Author and validate the contract

```bash
kiri contract new                       # Kiri drafts from the source shape
kiri contract validate contracts/sales/customer.yml --strict
```

`owner` and `kiri.classification` are **mandatory** — the canonical-model
validator fails without them. `--strict` is what your PR-time lint runs, so
green here means green in review. See
[contracts & the medallion model](medallion-and-contracts-model.md).

---

## 5. Plan — see what an apply would do

```bash
kiri plan --target prod
```

`plan` surfaces metadata conflicts and shows the bronze → silver work **without
touching a table**. Read it before every apply.

---

## 6. Apply — build bronze and silver live

```bash
kiri apply --target prod
# scope it while iterating:
kiri apply --target prod --contract customer
kiri apply --target prod --domain sales
```

`apply` runs the full pipeline — ingestion → AI description → contracts → dbt
build — against Databricks. Confirm the result:

```bash
kiri list-generated        # every file the generator owns
kiri silver plan           # the locked technique + planned silver contracts
kiri silver conformance    # technique-appropriate conformance + evidence
```

At this point you have a **live source endpoint and a governed silver layer** on
Databricks. See [plan, apply & debugging](plan-apply-and-debugging.md); when a
run fails, **`debug-apply`** has Kiri read
the error and propose the fix.

---

## 7. Build the gold layer

```bash
# draft the star schema, then apply it
kiri apply --target prod --domain sales      # once gold contracts exist
```

> **Let Kiri drive it:** **`draft-gold-model`**
> proposes a Kimball star (facts, dimensions, grain, conformed dimensions) from
> your silver. Localized gold view twins and the star-schema metadata are
> covered in [gold star schema & localized twins](gold-star-schema-and-localized-twins.md).

---

## 8. Schedule it — orchestration as native jobs

Turn the contracts into a running, scheduled pipeline compiled to native
Databricks jobs. `kiri compile` is a group — compile per (domain, layer), a
depth-range, or a single layer — then inspect the resulting DAG:

```bash
kiri compile layer --target prod    # contracts → platform-native workflow
kiri flows --target prod            # inspect the compiled DAG
```

The schedule (`kiri.schedule.cadence` on each contract) drives the job cadence.
See [orchestration & scheduling](orchestration-and-scheduling.md) for the full
set of `kiri compile` subcommands (`bronze` / `silver` / `layer` / `range`).

> **Let Kiri drive the whole arc:** **`databricks-build-pipeline`**
> orchestrates steps 2–8 end to end — endpoint → silver → gold → scheduled jobs —
> and then helps with day-2 additions.

---

## 9. Govern and promote to production

- **Catalog & governance** — annotations, PII, ownership flow into the catalog
  and can be reconciled against the live workspace. See
  [governance & catalog](governance-and-catalog.md) for the catalog and
  verification commands.
- **Promote** through the contract state machine (draft → active) and across
  environments with forward-only promotion + a release manifest
  ([release, promotion & CI/CD](release-promotion-and-cicd.md)):

  > **Let Kiri drive it:** **`promote-contract`**
  > advances a contract safely through the **ADR 0012**
  > lifecycle.

- **Compliance** — generate the DORA / EU AI Act / GDPR report from the same
  metadata + AI audit log: **`compliance-check`**
  and [security & compliance](security-and-compliance.md).

---

## You're in production

You now have: a live source endpoint, a governed bronze → silver → gold
medallion on Databricks, scheduled native jobs, a populated catalog, and a full
audit trail. From here, day-2 work is the same loop at smaller scope — add a
source (step 3), author its contract (step 4), `plan` / `apply`, promote.

## Related reference

- [Getting started walkthrough](getting-started-walkthrough.md) — the same loop on local DuckDB, zero cloud
- [Wiring MCP: Databricks and Kiri](mcp-databricks-and-kiri-setup.md) — drive all of this from a conversation
- [AI gateway setup](ai-gateway-setup.md) — the AI behind Kiri
- **`skills/`** — every reference skill referenced above
