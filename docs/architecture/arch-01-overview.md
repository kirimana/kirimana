# arch-01: Kirimana — Overview

Kirimana is an open-source, AI-native data contract platform. It treats contracts — not pipelines and not catalog entries — as the durable artefact of a data system, and uses those contracts to plan, apply, and verify what runs on the customer's data platform. This document describes what Kirimana is in its v0.9 private-preview form, who it is for, and how it is shipped.

## What Kirimana is

Kirimana is a control plane. It reads YAML data contracts written in the [Open Data Contract Standard (ODCS) v3](https://bitol.io), validates them, compiles them into platform-native objects (tables, jobs, schedules, lineage, quality rules), and emits a deterministic plan that a human or CI pipeline applies. The runtime work — query execution, ingestion, transformation — is done by the customer's existing platform. First-class platforms: Databricks, Microsoft Fabric, Trino + Iceberg + Polaris, DuckDB. MSSQL is supported as a migration source for legacy enterprise estates. Kirimana orchestrates compilation; the platform orchestrates execution.

The core is written in Python 3.12+, packaged as a uv-managed monorepo, and ships behind a thin CLI (`kiri`) plus a programmatic API (`dca-core`). A web application and an MCP server expose the same library functions through HTTP and AI-assistant integration surfaces respectively.

## Why it exists

Most data-governance products start from the catalog: discover what exists, then bolt policy onto it. Kirimana starts from the contract: write the intent, then generate everything that follows. The catalog is a derived view, not the system of record. This inversion matters because it makes contracts the only place a change can originate, which in turn makes audit, lineage, quality evidence, and compliance reporting fall out of one source rather than four.

Kirimana also takes a position on AI: language models are useful for drafting and explaining, never for unattended writes. The AI Gateway (arch-04) enforces a trust ladder that classifies every call and forbids autonomous mutation of governed assets.

## Two editions

- **Kirimana Databricks Edition** — the flagship build, target for enterprise customers running Databricks (Unity Catalog, Workflows, DQX integration). Production-ready for the use-cases listed in `/maturity`.
- **Kirimana Starter (OSS)** — the same core, configured for Trino + Iceberg + Polaris, DuckDB, or Postgres. Apache 2.0, designed for teams who want to evaluate, self-host, or run smaller workloads without a Databricks contract. Same contract format, same CLI, same federation model.

Both editions ship from one codebase. There is no closed-core split. Apache-2.0 applies to everything in `packages/` and `services/` (see arch-10).

## Audience

This documentation is written for data engineers and data architects evaluating Kirimana for a team that already runs a real warehouse and needs contracts, lineage, and audit evidence to hold up under review. It assumes familiarity with the medallion model, dimensional modelling, and the shape of a modern lakehouse. It does not assume any prior exposure to Kirimana.

## Status: v0.9 private preview

Kirimana is in private preview as of this document. The contract lifecycle, adapter model, AI Gateway, federation resolver, and compliance generators (DORA, EU AI Act, GDPR) are production-targeted; the proposed `aiPolicy` ODCS extension and several integration packs are still beta or experimental. Per-feature maturity is tracked on the public maturity page rather than in this document, so it doesn't drift.

License: Apache-2.0 for the core, in perpetuity. Commercial value lives in adjacent layers (managed service, compliance packs), never in feature flags on the open core.

## Related ADRs

001, 002, 004, 005, 006, 010, 013
