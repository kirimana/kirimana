# Changelog

All notable changes to the public Kirimana repository will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html) starting from v1.0.

## [Unreleased]

### Added
- Public manifesto: architecture docs, curated ADR set (001-032), governance files

## [0.9.0] — 2026-05 (Private Preview)

Kirimana enters private preview as an AI-native data contract platform built on ODCS v3.

### In this preview
- ODCS v3 contract model with `dca.*` extensions for AI policy, SLA, expectations, lineage hints
- AI Gateway with a 0/1/2 trust ladder ([ADR 016](docs/adr/016-ai-gateway-trust-ladder.md))
- Two-category adapter architecture (Platform + Framework)
- First-class platforms: DuckDB (local), Databricks (Azure), Microsoft Fabric, Trino + Iceberg + Polaris
- MSSQL adapter as a migration source
- MCP server for external AI agents ([ADR 017](docs/adr/017-mcp-as-external-ai-surface.md))
- Compliance evidence generators for DORA, EU AI Act, GDPR (artefact-as-input; human attestation still required)
- Multi-environment CI/CD with git-SHA as ground truth ([ADR 032](docs/adr/032-multi-environment-cicd.md))

Design-partner access is invite-only. See [kirimana.io](https://kirimana.io) for the early-access form.
