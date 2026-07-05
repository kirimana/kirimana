# Contributing to Kirimana

Thank you for your interest. Kirimana is built on the belief that platform-agnostic, governed data automation should be a community effort — not a proprietary product.

> **Kirimana is in an invitation-only private beta (1.0.0-beta.1).** The engine source lives in a private repository during the beta, so code contributions are handled through the beta program rather than this public front-door repo. **Request beta access at [kirimana.io](https://kirimana.io).** This repository (the public front-door: README, CLI docs, community-health files) accepts documentation fixes and issues directly.

## Ways to contribute

1. **Industry packs** — the highest-leverage contribution. Build a reusable data model for a common source system (Microsoft Dynamics, Salesforce, SAP, Workday, …).
2. **Adapters** — add support for a new target platform (Snowflake, BigQuery, Redshift, Postgres) or a new transformation framework (SQLMesh, Dagster, Airflow).
3. **Core** — canonical model, ODCS parser, AI service layer, catalog, CLI.
4. **Documentation** — tutorials, architecture deep-dives, translations. Fixes to the [CLI docs](docs/cli/) in this repo are welcome as PRs.

## Developer Certificate of Origin (DCO)

Every commit must carry a `Signed-off-by:` line (`git commit -s`). By adding the sign-off, you certify the terms of the [Developer Certificate of Origin](https://developercertificate.org/). No CLA is required.

## Principles we hold contributions to

Every PR is evaluated against the project's six design principles. In short:

- **Flow.** Change must trace to a user need. Describe the persona and journey in the PR.
- **AI-first + ownership.** New contracts and metadata objects declare `owner` and `kiri.classification`.
- **Simplicity.** Prefer existing tools (dbt, DuckDB) over forking them. Complexity needs justification.
- **Independence.** Platform-specific code lives behind the adapter interface. No platform-specific features in core.
- **Modularity.** Services communicate via documented contracts (OpenAPI / Pydantic). No cross-service imports.
- **Security.** Security/governance is designed in, not added later. Every AI call is audit-logged.

## Pull request checklist (documentation PRs in this repo)

- [ ] Every commit signed off with DCO (`git commit -s`)
- [ ] Links resolve (no dead paths)
- [ ] Prose matches the private-beta framing (no self-host / source instructions that don't apply to this repo)

Engine, adapter, and pack contributions follow the private-beta program's own checklist — request access at [kirimana.io](https://kirimana.io).

## Discussions & governance

- Questions & ideas: GitHub Discussions
- Bugs & features: GitHub Issues
- Security: see [SECURITY.md](SECURITY.md)
- Governance: see [GOVERNANCE.md](GOVERNANCE.md)
