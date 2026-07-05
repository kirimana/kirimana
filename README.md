# Kirimana

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[![Python 3.12+](https://img.shields.io/badge/python-3.12%2B-blue.svg)](https://www.python.org/downloads/)
[![PyPI](https://img.shields.io/pypi/v/kiri-cli.svg)](https://pypi.org/project/kiri-cli/)

**Turn a data contract into a running, governed data platform — on the runtime you already use.**

*Kirimana* is Māori for **contract** — literally, to place your *mana* (honour, authority) on paper. You describe each dataset once, as a contract. Kirimana generates the pipelines, enforces the governance, and keeps the whole thing traceable — with an AI assistant, **Kiri**, doing the heavy lifting when you want it and staying out of the way when you don't.

> **Status: private beta — invitation only (1.0.0-beta.1).** The engine, the `kiri` CLI, and the DuckDB and Databricks runtimes work end-to-end today. The `kiri-cli` wheel is published to PyPI; the repository and support are invitation-gated. Interfaces still move between beta releases — this is not GA. **Request access at [kirimana.io](https://kirimana.io).**

Open source · Apache-2.0 · community-driven.

---

## What is Kirimana?

Most teams building a data platform face the same fork in the road:

- **Buy into one vendor** — Delta Live Tables, Fabric Dataflows — and accept the lock-in, or
- **Hand-assemble a dozen open-source tools** — ingestion, transformation, catalog, quality, orchestration, lineage — and own the glue forever.

Kirimana takes a third path. **You declare each dataset as a data contract** — an [ODCS](https://bitol.io/) document that states, in one place, where the data comes from, what it looks like, who owns it, how it's classified, how often it runs, and what quality it must meet. From that contract Kirimana:

1. **Generates the transformations** (bronze → silver → gold) as plain dbt-core models,
2. **Runs them on your platform** through an adapter — DuckDB on your laptop, Databricks in the cloud,
3. **Enforces governance** — owner, classification, PII, retention, lawful basis — in the canonical model *and* the catalog, and
4. **Keeps a full audit trail** of every apply and every AI call.

The contract is the source of truth. Everything downstream is generated from it and traceable back to it.

## Who it's for

**Enterprise-grade by design, accessible to every team.** The same contract that governs a two-person analytics team scales to a regulated enterprise — the guarantees (ownership, classification, lineage, audit) only compound as the estate grows. Kirimana is for teams who want the rigour of a governed platform without a platform team's worth of glue code.

## What a contract looks like

A contract is ordinary ODCS with a documented `kiri.*` extension namespace for the things a data platform needs to know:

```yaml
apiVersion: 3.0.0
kind: DataContract
id: com.example.crm.customer
name: CRM Customer
version: 1.0.0
domain: crm
owner: crm-team@example.com

schema:
  - name: customer
    logicalType: table
    properties:
      - name: customer_id
        logicalType: string
        primaryKey: true
        required: true
      - name: email
        logicalType: string
        required: true
        unique: true
        customProperties:
          - property: kiri.pii.categories
            value: [contact]
          - property: kiri.pii.direct_identifier
            value: true

customProperties:
  - property: kiri.classification
    value: confidential
  - property: kiri.gdpr.lawful_basis
    value: contract
  - property: kiri.gdpr.retention_period_days
    value: 2555
  - property: kiri.schedule.cadence
    value: daily
```

That single document drives code generation, the governance catalog, the quality checks, the schedule, and the compliance view — no second config to keep in sync.

## What makes it different

- **Contract-first, AI-readable.** One ODCS document answers *everything* about a dataset. It's designed to be read by humans, tools, and AI alike.
- **AI-augmented, never AI-dependent.** Kiri can draft contracts, onboard a source from a plain-English description, and explain a failed run — but every AI call goes through a trust-laddered gateway with audit, cost control, and an air-gapped local-model option. Turn the AI off and the platform works identically, just with less acceleration.
- **Governance is enforced, not documented.** Owner and classification are mandatory. PII, lawful basis, and retention travel with the data into the catalog. Secrets are never in code or YAML — only vault references.
- **PR-time linting.** Contract violations are caught in review, before they reach a warehouse.
- **Portable by construction.** The transformations are plain dbt-core; the platform-specific bits live behind adapters. Your logic isn't trapped in a proprietary runtime.
- **Everything as code.** Contracts, policy, schedules, and releases are versioned files — reviewable, diffable, and reproducible.

## Quick start

Install the `kiri` CLI from PyPI (**Python 3.12+**):

```bash
pip install kiri-cli          # base + local DuckDB runtime
# optional runtime extras: kiri-cli[databricks], [trino], [iceberg], [all]

kiri --version
kiri init my_warehouse --silver-technique flat
cd my_warehouse
kiri --help                   # explore: init · ingest · contract · plan · apply · catalog · …
```

`kiri init` scaffolds a self-contained data-warehouse repository. From there you add sources, author contracts (by hand or with Kiri's help), then `kiri plan` to preview and `kiri apply` to build. A guided setup for a Databricks target ships as a reference workflow.

See the full command reference in [`docs/cli/`](docs/cli/).

### Point your assistant at it (MCP)

Kirimana has **no web UI — your LLM assistant is the interface.** The `kiri-mcp` server (ships with `kiri-cli`) exposes your project's catalog, lineage, and PII to any MCP-capable client (Claude Desktop, Claude Code, Cursor, …). For Claude Desktop, add to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "kiri": {
      "command": "kiri-mcp",
      "env": { "KIRIMANA_PROJECT_DIR": "/absolute/path/to/my_warehouse" }
    }
  }
}
```

Then just ask Kiri: *"Using the kiri catalog, what PII do we hold — and draft a contract for the orders table."* Kiri reads your project, drafts the artifacts, and hands you the `kiri plan` / `kiri apply` commands to run.

### Let Kiri drive it — reference skills

Kirimana ships a set of **Claude Code skills** — small, governed wrappers around the CLI that codify the governance rules. Invoke with `/<skill-name>` in Claude Code, or ask your MCP-connected assistant to run one. Good starting points:

- **`databricks-project-setup`** — guided end-to-end setup of a new project on Databricks (sizing, scaffold, naming, target + secrets)
- **`databricks-build-pipeline`** — build a full pipeline end to end: endpoint → silver → gold → scheduled jobs
- **`add-source`** — onboard a new ingest source (airbyte / native / dlt / debezium / landing_zone)
- **`draft-gold-model`** — draft a Kimball star schema from your silver layer
- **`promote-contract`** — advance a contract through the medallion state machine
- **`debug-apply`** — have Kiri read a failed run and propose the fix

The skills ship with the CLI and source. **New here?** Follow the runbook: [**From first startup to production**](docs/cli/guides/first-platform-to-production.md) — `pip install` → live endpoint → full medallion on Databricks. See also [wiring MCP](docs/cli/guides/mcp-databricks-and-kiri-setup.md) and [AI-gateway setup](docs/cli/guides/ai-gateway-setup.md).

### Source access (private beta)

Kirimana is in an **invitation-only private beta**. The engine source, the self-host and adapter tooling, and the runnable examples live in a private repository during the beta and are not part of this public front-door repo.

The published `kiri-cli` wheel on PyPI is the primary way to use Kirimana today (see the quick start above). For source access, self-hosting, or to join the beta, **request access at [kirimana.io](https://kirimana.io).**

## Architecture at a glance

```
        You  —  "What PII is here?  Draft the orders contract.  Why did silver fail?"
              │   plain language
   ┌──────────▼──────────────────────────────────────┐
   │  Claude (Desktop / Code)  ·  or your preferred LLM │   ← your interface: a conversation
   └──────────┬──────────────────────────────────────┘
              │   MCP  ·  kiri-mcp  (catalog · lineage · PII · contracts)
   ┌──────────▼──────────────────────────────────────┐
   │  kiri  —  the CLI (Kiri, the AI assistant)        │   ← Kiri drafts · you plan / apply
   └──────────┬──────────────────────────────────────┘
              │
   ┌──────────▼──────────────────────────────────────┐
   │  Engine — platform-agnostic core:                 │
   │  contract model · code generation · AI gateway ·  │
   │  quality · catalog · vault                         │
   └──────────┬──────────────────────────────────────┘
              │   adapters (platform + framework)
   ┌──────────▼──────────────────────────────────────┐
   │   DuckDB (local)        Databricks (Delta)        │   ← runtimes exercised today
   └──────────────────────────────────────────────────┘
```

**Your interface is a conversation — there's no web UI, and you don't need one.** Kirimana ships an **MCP server** (`kiri-mcp`), so you point Claude (Desktop or Code) — or any preferred LLM that speaks the [Model Context Protocol](https://modelcontextprotocol.io) — at your project and just describe what you want: *"what PII is in this catalog?"*, *"draft a contract for the orders table"*, *"what does the silver layer look like, and why did the last run fail?"* **Kiri**, the assistant persona, reads your contracts, catalog, and lineage through MCP, drafts the artifacts, and hands you the exact `kiri plan` / `kiri apply` commands to run — every AI call audited through the gateway. Prefer the keyboard? Every capability is a plain `kiri` subcommand you run yourself; the assistant is an accelerator on the same engine, never a dependency.

The core is deliberately platform-agnostic: transformations are generated as dialect-rendered dbt-core, and everything platform-specific sits behind a documented adapter seam. **DuckDB and Databricks are the runtimes exercised today.** The adapter architecture is how additional platforms are added — that surface exists, but only these two are shipped and tested.

Industry data models and paid capabilities ship as installable **packs**, so the open-source core stays lean.

## Learn more

| Where | What |
|---|---|
| [`docs/cli/`](docs/cli/) | The full `kiri` CLI reference and how-to guides |
| [`docs/cli/getting-started.md`](docs/cli/getting-started.md) | From `pip install` to your first built warehouse |
| [kirimana.io](https://kirimana.io) | Product overview, the design principles, and private-beta access |

## Contributing

Contributions are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for the workflow (conventional commits, DCO sign-off, tests on DuckDB), [GOVERNANCE.md](GOVERNANCE.md) for how decisions are made, [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) for community expectations, and [SECURITY.md](SECURITY.md) to report a vulnerability.

## License

[Apache-2.0](LICENSE) — for the core, forever. Copyright notice in [NOTICE](NOTICE).
