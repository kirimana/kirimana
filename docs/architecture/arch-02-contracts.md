# arch-02: Contracts as the System of Record

Kirimana's central design choice is that an ODCS v3 data contract is the canonical metadata artefact for a dataset. Everything else — physical tables, jobs, lineage, quality rules, catalog entries, compliance evidence — is generated, regenerated, or validated against the contract. This document describes the contract format, its lifecycle, and the plan/apply/verify model that turns a YAML file into a governed dataset on a target platform.

## ODCS v3 as canonical format

Kirimana adopts the Open Data Contract Standard v3, published by the Linux Foundation's Bitol project, as its native contract format. We do not maintain a Kirimana-specific schema and we do not transcode ODCS into an intermediate representation. The YAML the user writes is the YAML the planner reads.

Where Kirimana needs metadata that ODCS v3 does not yet model (medallion zone assignment, lineage URNs, dimensional roles, scheduling, AI policy), we use `customProperties` under a reserved `dca.*` namespace. Examples: `dca.medallion.layer`, `dca.lifecycle.state`, `dca.medallion.silver_zone`, `dca.attribute.review_state`, `dca.schedule.cron`, `dca.lineage.from_columns`. These namespaces are documented and stable; canonical paths are versioned and old aliases follow a four-step deprecation cadence (parser accepts both → lint WARN → lint ERROR → remove).

**A note on the `dca.*` customProperties namespace.** The prefix derives from Kirimana's internal codebase namespace. A migration path to `kirimana.*` with `dca.*` as a deprecated alias is planned ahead of v1.0; existing contracts will continue to validate.

When an extension is broadly useful — `aiPolicy` for governing what AI assistants may read or write against a dataset — we propose it upstream to the Bitol working group rather than fork (see arch-10).

## Contract lifecycle and state machine

Every contract carries a lifecycle state on the `dca.lifecycle.state` customProperty:

- `draft` — author is writing, no apply allowed against shared environments
- `active` — contract is the binding spec for the dataset in one or more environments
- `deprecated` — still readable, no new dependencies allowed
- `retired` — historical record only; tables may still exist but contract is not authoritative

Transitions are not free-form. They run through the same plan/apply/verify pipeline as any other contract change, which means they appear in audit (arch-06), trigger lineage updates (arch-06), and respect environment cursors (arch-08). Deprecation in particular blocks downstream contracts that still bind to the retired column or table — federation (arch-05) makes that block enforceable across projects.

Source contracts and consumer contracts are split. A physical source (a Kafka topic, an Airbyte destination table) has one source contract; many consumer contracts may bind to it. Bindings are URN-based and resolved at plan time.

## Plan / apply / verify

Kirimana follows a Terraform-shaped lifecycle on top of contracts:

- **plan** reads the contracts and current platform state through the adapter (arch-03), produces a deterministic, byte-stable diff describing what would change, and writes it to disk. Plans are signed by the contract content hash plus the adapter conformance version (arch-10); regenerating from the same inputs produces the same output, byte for byte.
- **apply** executes the plan against the target via the adapter. Apply is idempotent: re-running the same plan on a consistent target is a no-op. Apply emits audit events (arch-06) and updates the catalog (arch-05).
- **verify** re-reads platform state and checks invariants — table shape matches contract, lineage references resolve, quality engine has run where required (arch-07). Verify is the gate compliance generators evaluate against.

Because plans are deterministic and apply is idempotent, promotion across environments (dev → test → prod) is a forward-only sequence of plans against environment cursors rather than a re-derivation per environment. Each environment moves forward independently; rollbacks are forward by reverting the contract change, not by reversing applied state.

## Validation

`kiri validate` runs a layered lint pass:

- ODCS v3 schema conformance (delegated to the Bitol validator)
- Kirimana semantic checks: medallion-layer invariants, naming conventions, silver-zone leaf invariants, dimensional-model lint (Kimball star rules), lineage resolvability
- Adapter-specific checks via the active PlatformAdapter (e.g. Databricks identifier rules, Iceberg partitioning constraints)

Validation runs locally, in CI, and as a precondition of every apply. It is the same code path in all three places.

## Related ADRs

006, 007, 008, 009, 010, 012, 023
