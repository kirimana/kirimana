# arch-10: Standards and Conventions

Kirimana is built on open standards where they exist and contributes back when they don't yet cover what's needed. The contract format is ODCS v3 from the Linux Foundation's Bitol project; the lineage wire format is OpenLineage; the external AI surface is the Model Context Protocol. Internal conventions — naming, versioning, conformance — are documented and stable. This document inventories the standards, the proposed extensions, and the conventions that govern day-to-day Kirimana use.

## ODCS v3 (Bitol / LF AI & Data)

The [Open Data Contract Standard v3](https://bitol.io) is the canonical contract format. Bitol sits under the Linux Foundation's AI & Data umbrella; Kirimana adopts the spec as published and validates against the published JSON Schema as a precondition of every plan and apply.

Where Kirimana needs metadata ODCS doesn't yet model, the convention is `customProperties` under a reserved `dca.*` namespace. Documented namespaces:

- `dca.lifecycle.state` — lifecycle state machine (arch-02)
- `dca.medallion.layer`, `dca.medallion.silver_zone` — medallion semantics (arch-02)
- `dca.attribute.review_state` — per-column review state machine
- `dca.schedule.*` — orchestration scheduling (canonical; `dca.orchestration.schedule.*` is a deprecated alias)
- `dca.lineage.from_columns` — column-level lineage URNs (arch-06)
- `dca.domain.*` — domain scoping (arch-05)
- `dca.gdpr.legal_basis`, `dca.retention.*` — compliance metadata (arch-07)

Canonical paths are versioned. When a namespace migrates (as `dca.medallion.state` did to `dca.lifecycle.state`), a four-step cadence applies: parser accepts both → lint WARN → lint ERROR → remove. Two minor versions minimum between steps.

## Proposed: `aiPolicy` extension to ODCS

Kirimana has proposed an `aiPolicy` extension to the Bitol working group. It models three things:

- **trainingData** — whether the data may be used to train models, with vendor scope (e.g. "internal only", "third-party allowed under DPA")
- **piiExposure** — declared PII presence at the property level, drives gateway redaction (arch-04)
- **egressRules** — what AI providers and which classes of operation may see this contract

The proposal is to make these first-class ODCS fields rather than vendor extensions. Kirimana ships a reference implementation under `dca.ai_policy.*` today; the proposal is for the namespace to lift into the standard so other tools (catalog vendors, governance products) can read and write the same metadata.

Status: proposed to the Bitol WG; not yet accepted. Kirimana continues to ship the reference implementation regardless of the WG timeline.

## Conformance test suite for adapters

Every adapter category (PlatformAdapter, FrameworkAdapter, OrchestrationAdapter, CatalogBackend) has a conformance suite shipped from `packages/adapters/tests/`. The suite is a versioned set of canonical contracts plus expected plan and apply outcomes. An adapter declares the surface version it implements; the suite runs the contracts at that version and reports pass/fail.

Conformance is a precondition of registry eligibility for third-party adapters and a precondition of release for first-party adapters. The conformance result is part of the plan signature (arch-02) — plans are not portable across adapter versions even on identical input.

## Naming conventions

Project-level naming is configured in `dca.yml` and locked at first silver write. The convention covers:

- **Data Vault entities** — `hub_*`, `link_*`, `sat_*` prefixes; configurable
- **Medallion zones** — `bronze_*`, `silver_*`, `gold_*` schema prefixes; configurable
- **Meta columns** — `_dca_*` reserved namespace for Kirimana-managed metadata columns (load timestamps, source-system identifiers, trace IDs)
- **URN scheme** — `urn:kirimana:contract:<domain>.<dataset>:<version>` and `urn:kirimana:column:<domain>.<dataset>:<version>#<column>`

The reserved `_dca_` column prefix is non-negotiable; the rest is configurable per project but locked once silver is written so cross-environment promotion stays deterministic.

## Semantic versioning of contracts

Contracts version under SemVer with data-contract semantics:

- **MAJOR** — breaking schema change (column removed, type narrowed, semantics changed)
- **MINOR** — additive change (column added, constraint relaxed)
- **PATCH** — metadata or documentation change

Consumers bind to a specific MAJOR.MINOR and accept PATCH updates automatically. MAJOR changes require either consumer-side update or a parallel-version window (the producer publishes 1.x and 2.x simultaneously for a deprecation period). Federation (arch-05) enforces this — a cross-project consumer cannot resolve a deprecated version past its retire date.

Surface versioning for adapters (arch-03) follows the same SemVer discipline but with its own deprecation cadence (two MINOR releases between surface-removal phases).

## Open standards Kirimana relies on

- **ODCS v3** — contracts
- **OpenLineage** — lineage events
- **Model Context Protocol** — external AI integration
- **OIDC** — identity and workload identity (arch-08)
- **Apache Iceberg** — open table format
- **Apache 2.0** — license for the Kirimana core, in perpetuity

Standards adoption is a long-term commitment, not a v0.9 expedient. Where a standard exists and is healthy, Kirimana uses it; where one is missing, Kirimana proposes one.

## Related ADRs

001, 006, 007, 008, 012, 014, 017, 023
