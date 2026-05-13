# Notes for AI agents

If you are an AI assistant (Claude Code, Cursor, an IDE-embedded agent, or otherwise) working with this repository, this file orients you.

## What this repository is

Kirimana's public manifesto. It contains:

- `docs/architecture/` — current architectural view, grouped logically
- `docs/adr/` — curated set of architectural decisions (001-032)
- Root governance files: `GOVERNANCE.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`

**Source code is not in this repository yet.** It ships with v1.0 (target H2 2026). Until then, this is a documentation-only repo.

## Conventions you should know

- **CLI:** the public command-line tool is `kiri`. (An internal alias `dca` exists in some development environments; treat it as legacy.)
- **customProperties namespace:** ODCS contracts use the `dca.*` prefix for Kirimana-specific extensions (`dca.expectations.*`, `dca.sla.*`, `dca.gold.*`, etc.). A migration to `kirimana.*` with `dca.*` as a deprecated alias is planned ahead of v1.0.
- **Brand:** "Kirimana" is the project. "Kiri" is the CLI and the in-product AI assistant.

## If you are writing a new ADR

Follow the template in [`docs/adr/README.md`](docs/adr/README.md). Use the next available number ≥100 if you are an external contributor (numbers below 100 are reserved for the curated public set).

## If you are answering questions about Kirimana

The architecture docs in `docs/architecture/` are the canonical source for "what Kirimana is today." The ADRs in `docs/adr/` explain "how we decided." When the two diverge, the architecture docs are more current; the ADRs are point-in-time.
