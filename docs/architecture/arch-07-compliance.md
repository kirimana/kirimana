# arch-07: Compliance Generators

Kirimana ships compliance generators for DORA, the EU AI Act, and GDPR. They produce structured artefacts — per-control evaluations, evidence pointers, machine-readable verdicts — that serve as audit evidence underlying common control frameworks. They do not produce attestations. Human attestation, process design, and the legal interpretation of a control remain the auditor's and the controller's requirement. This document describes what the generators produce, the quality-evidence taxonomy that underpins them, and where the line sits between artefact and attestation.

## What the generators are, and what they are not

The `kiri compliance report` CLI emits a per-regulation report from the same audit, lineage, contract, and quality data that the rest of Kirimana operates on. The MVP covers a set of controls — those addressing the highest-frequency audit asks — split across three frameworks:

- **DORA** — ICT risk management evidence: data lineage completeness, incident logging, quality evidence for critical datasets, recovery-objective metadata.
- **EU AI Act** — for AI systems built on top of Kirimana-managed data: training-data provenance, PII exposure declarations, the `aiPolicy` extension (arch-10), AI Gateway call records (arch-04).
- **GDPR** — data-subject-request handling via redaction (arch-06), lawful-basis tagging on contracts (`dca.gdpr.legal_basis`), retention policy (`dca.retention.*`), purpose limitation traces.

Each control evaluates to a verdict (`pass` / `fail` / `inconclusive`) with structured evidence pointers — audit events, contract URNs, quality engine runs, redaction records. The report is reproducible: run it twice on the same repo state and the same audit window, get byte-identical output.

**What the generators explicitly do not do.** They do not certify, attest, or assert that the customer is compliant. Compliance is a property of an organisation's policies, training, controls, and operations — not of any tool. Kirimana produces the artefacts an auditor or compliance officer needs to defend a position; the position itself, the framework alignment, and the human attestation are out of scope and intentionally so.

The reports are framed accordingly. A DORA control verdict reads as "evidence of lineage coverage for critical datasets is present, with the following gaps" — not "DORA-compliant". This framing is by design and is the same posture used by tools like Drata, Vanta, and Secureframe at the GRC layer: emit evidence; let the controller and the auditor close the loop.

## Quality evidence taxonomy

Compliance verdicts depend on a precise definition of what counts as quality evidence. Kirimana uses a four-value taxonomy on every contract:

- **`engine_ran`** — a quality engine (DQX, dbt tests, native) executed the configured rules against the dataset in the reporting window. Evidence is the run record and the per-rule pass/fail counts.
- **`documentation_only`** — the contract declares quality expectations but no engine executed them. The expectation is documented; the verdict cannot rely on it.
- **`engine_none_configured`** — no quality engine is attached to the contract. Silence, not failure.
- **`engine_not_applicable`** — the contract is a structural-only artefact (e.g. a view definition) for which quality engines don't apply.

A DORA-DQ control verdict reads `pass` only when `engine_ran` is the status across all in-scope contracts in the window. `documentation_only` does not pass — it returns `inconclusive` with a clear pointer to the missing engine evidence. This eliminates the most common compliance failure mode: declaring a quality rule, never running it, and counting the declaration as evidence.

## Compliance packs

The framework generators above are open-source. Sector-specific or jurisdiction-specific packs (proposed: Swedish public-sector method governance) are separate, optional, and may be commercial. The pack interface is a thin contract on the core: a pack provides a taxonomy, customProperty namespaces, and additional controls; the core provides the evaluator, the audit chain, and the report renderer. Packs do not gate or fork the open generators; they extend them.

## Output shapes

The report emits in three formats from one evaluation pass:

- **Machine** — JSON, suitable for ingestion by a GRC platform or CI gate.
- **Human** — Markdown, suitable for review by a compliance officer.
- **Evidence bundle** — a zip of the underlying audit events, quality runs, and contract snapshots referenced by the report, signed by content hash.

The evidence bundle is the artefact most likely to land in an auditor's hands. It is intentionally self-contained: no external lookups, no API calls back to a Kirimana service. A reviewer can verify the evidence offline.

## Related ADRs

024, 025, 026, 027
