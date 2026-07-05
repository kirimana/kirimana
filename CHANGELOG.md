# Changelog

All notable changes to Kirimana are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project aims to follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
Pre-release versions use PEP 440 spelling for the published `kiri-cli` wheel
(`1.0.0b1` == `1.0.0-beta.1`).

## [Unreleased]

## [1.0.0-beta.1] - 2026-07-04

First public beta of Kirimana — an **invitation-only private beta**. The
`kiri-cli` wheel is published to PyPI; the repository and support are
invitation-gated. This release aligns every workspace package to a single
`1.0.0b1` version and establishes the release-engineering baseline.

### What works today

- **Contract-driven medallion generation.** Author each dataset once as an
  ODCS v3 data contract with the documented `kiri.*` extension namespace;
  Kirimana generates bronze → silver → gold as plain dbt-core models and
  keeps them traceable back to the contract.
- **Two exercised runtimes.** DuckDB for local, no-provisioning builds and
  Databricks (Delta) for the cloud. Additional platforms are an architectural
  adapter surface, not yet shipped runtimes.
- **Governance and catalog.** Owner and classification are mandatory; PII,
  lawful basis, and retention travel with the data into a pluggable
  governance catalog. Column-level lineage is carried as contract metadata.
- **Data quality.** Contract-derived quality checks (Great Expectations) and
  a silver-quality report.
- **AI-assisted flows via the audited gateway.** Kiri can draft contracts,
  onboard a source from a plain-English description, and explain a run — every
  LLM call routes through the trust-laddered AI gateway with prompt, response,
  model, cost, and caller logged. An air-gapped local-model provider is
  available, and the platform works identically with the AI turned off.
- **The `kiri` CLI.** `init`, `ingest`, `contract`, `plan`, `apply`,
  `catalog`, and more. Installable from PyPI as `pip install kiri-cli`
  (Python 3.12+).

### Notes

- Interfaces may still move between beta releases; this is not a GA release.
- Distribution is a self-contained fat wheel — `pip install kiri-cli` needs no
  other PyPI projects. See `docs/RELEASING.md`.

[Unreleased]: https://github.com/dbarton1974/kirimana/compare/v1.0.0-beta.1...HEAD
[1.0.0-beta.1]: https://github.com/dbarton1974/kirimana/releases/tag/v1.0.0-beta.1
