# Kirimana

> **Open source AI-native data contract platform.**
> Data engineers and data architects turn ODCS v3 contracts into compiled, runnable workflows across Databricks, Fabric, Trino, DuckDB and other SQL-bearing platforms — with AI policy gating built in from the first line.

---

## Status

**Private preview, v0.9 (2026-05).** Source code goes public alongside v1.0, targeted for H2 2026. This repository currently holds the manifesto: architecture, decisions, governance.

If you'd like early access as a design partner, see [kirimana.io](https://kirimana.io).

## What you'll find here

- **[`docs/architecture/`](docs/architecture/)** — current view of what Kirimana is, grouped logically. Start with [`arch-01-overview.md`](docs/architecture/arch-01-overview.md).
- **[`docs/adr/`](docs/adr/)** — curated set of 32 architectural decision records.
- **[`GOVERNANCE.md`](GOVERNANCE.md)** — who stewards the project, foundation path, licence commitment.
- **[`SECURITY.md`](SECURITY.md)** — how to report vulnerabilities.
- **[`CONTRIBUTING.md`](CONTRIBUTING.md)** — what we can accept today, what opens up at v1.0.

## What you won't find here yet

The engine, adapters, CLI, AI Gateway, web UI, and Helm chart. All of that lands when the source code goes public at v1.0.

## Brand and licence

- **Licence:** [Apache-2.0](LICENSE), forever. No BSL. No CLA. DCO sign-off only.
- **Steward:** [David Barton Consulting AB](https://kirimana.io) (Sweden). Foundation path described in [GOVERNANCE.md](GOVERNANCE.md).
- **Standards:** Built on [ODCS v3](https://bitol-io.github.io/open-data-contract-standard/) (Bitol / Linux Foundation AI & Data).

## Read order for newcomers

1. [`docs/architecture/arch-01-overview.md`](docs/architecture/arch-01-overview.md) — what Kirimana is in 5 minutes
2. [`docs/adr/001-apache-2-0-license-forever.md`](docs/adr/001-apache-2-0-license-forever.md) and [002](docs/adr/002-governance-and-stewardship.md) — why we built it open
3. [`docs/adr/006-odcs-v3-canonical-metadata.md`](docs/adr/006-odcs-v3-canonical-metadata.md) and [011](docs/adr/011-two-category-adapter-model.md) — the two load-bearing architectural choices
4. [`docs/architecture/arch-04-ai-gateway.md`](docs/architecture/arch-04-ai-gateway.md) — what makes this AI-native, not AI-bolted-on
