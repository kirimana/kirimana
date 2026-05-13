# ADR 026 — Compliance generators as audit-evidence framework

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** original ADR 0032 (archived)

## Context

Enterprise buyers — especially in finance, health, and EU-regulated industries — ask variants of the same question in their first meeting:

> "When the auditor shows up, can you hand them evidence that your platform is DORA / GDPR / EU AI Act compliant?"

Kirimana already records the raw material: AI audit log, MCP audit log, RBAC bindings, OpenLineage events, apply log, attribute review-state history, and contract classification metadata. What is missing is a single artefact that *maps* this material to the control languages auditors use. Without it, a compliance officer has to grep seven files and read seven ADRs to answer one question.

**Honest framing:** compliance generators produce structured artefacts that serve as evidence underlying control frameworks. They are not a legal opinion and they do not replace human attestation. They reduce the manual labour of evidence assembly; the auditor and the customer's compliance officer still own interpretation and sign-off.

Three options considered:

1. **External GRC tool of record** (Drata, Vanta) as the only story. Rejected — GRC tools cover infrastructure, HR, and policies but their data-platform coverage is shallow. Kirimana still needs to speak the control language for the slices only it owns.
2. **Free-form markdown audit doc per framework.** Rejected — hand-written docs rot faster than the regulations and provide no CI hook.
3. **Generated report with a curated control → evidence mapping, CLI-driven, JSON + Markdown.** Accepted.

## Decision

`kiri compliance report` generates a structured evidence artefact per framework. JSON is the primary format (machine-consumable, easy for GRC import); Markdown is for humans.

```bash
kiri compliance report --framework dora|ai_act|gdpr \
    [--period 2026-01-01..2026-12-31] \
    [--format json|markdown] \
    [--project <dir>]
```

**Scope — three frameworks in MVP:** DORA (mandatory for EU financial entities), EU AI Act (in force through 2026-2027 for high-risk AI systems), and GDPR Articles 5, 9, 25, 30, 32. ISO 27001, SOX, HIPAA, and NIS2 follow the same pattern but are out of scope for MVP.

**Per-control evaluation shape:** `control_id`, `framework`, `title`, `status`, `evidence_sources`, `sample_count`, `details`, `notes`. Status semantics are deliberate:

- `pass` — all evidence found, all invariants met.
- `partial` — evidence exists but an invariant is weakly met (e.g. 90% classification coverage).
- `fail` — clear violation found in the data.
- `inconclusive` — no data available in the requested period. **NOT the same as pass** — calls out that the auditor cannot trust the control based on what Kirimana has seen.

The `inconclusive` status is the key honesty mechanism. Silent passes are a lie of omission; compliance officers hate being surprised.

**MVP control registry (13 controls):** four for DORA (classification coverage, third-party AI access, ICT incident reporting, resilience testing), four for EU AI Act (transparency, human oversight, data governance, risk management / RBAC), five for GDPR (data minimisation, special categories, privacy by design, records of processing, security). Each maps to existing sources — no new instrumentation needed.

**Invariants:** the collector is **read-only** on audit streams, catalog, and project config. Evidence is hash-referenced where raw content is sensitive — e.g. a GDPR Article 9 finding references the column URN and a SHA of the classification line, not the classification bytes. Period filter applies to time-stamped sources; non-temporal sources are noted as "current state".

**Disclaimer is mandatory in every report header:** "generated evidence summary, not a legal opinion. Legal interpretation and final attestation remain the customer's and auditor's responsibility." This framing is non-negotiable.

What does NOT change: existing audit logs stay in JSONL; GRC integration is via JSON export, no bidirectional sync; the report carries the customer's framework interpretation responsibility.

## Consequences

- **Positive:** One command answers the most common enterprise first-meeting question with structured, reproducible evidence rather than vendor marketing. CI gate ready — `report --framework gdpr | jq '.summary.fail == 0'` can fail the build. Honest about gaps via `inconclusive`. Extensible — adding ISO 27001 is one framework enum value plus ~10 control functions.
- **Negative / costs:** Control curation is ongoing — EU regulations mutate. Mitigated by small MVP control count and per-control versioning. Evidence-source coverage is uneven across controls (some map to strong sources, others to weak ones); the `notes` field spells out scope. No long-term evidence retention — the report reads current logs, so out-of-window periods return `inconclusive`. The report is **not a legal opinion** and **does not replace human attestation**; misuse as if it were is a real risk that the disclaimer mitigates but cannot eliminate.
- **Neutral:** The report is stateless and regenerated each time; period-to-period comparison is by running twice and diffing JSON.

## Related

- See [arch-07-compliance](../architecture/arch-07-compliance.md)
- Related ADRs: 016 (AI Gateway trust ladder), 024 (audit redaction), 025 (quality evidence)
