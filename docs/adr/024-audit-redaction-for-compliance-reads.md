# ADR 024 — Audit redaction for compliance reads

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** original ADR 0037 (archived)

## Context

Two audit logs ship today: the AI gateway log (one JSONL line per LLM call) and the MCP log (one line per resource read or tool call). Both are deliberately **append-only** — any operator who can rewrite the log can hide AI calls that read classified data, defeating the traceability claim and the compliance posture.

Append-only is, however, incompatible with three real legal needs:

1. **GDPR Article 17 (right to erasure)** — a data subject's identifier ends up in a prompt and the regulator orders removal. We cannot legally retain personal data indefinitely.
2. **EU AI Act and DORA** — both require demonstrating *that audit happened*, not retaining byte-for-byte content. A redacted record proves the call took place under known policy.
3. **Data-residency rotation** — when a customer migrates regions, lingering personal data in the source-region audit log is itself a residency violation.

A naive `audit purge` that hard-deletes the row is wrong on every axis — it tampers with the audit trail and gives hostile actors a one-shot wipe.

## Decision

Introduce `kiri audit redact` — overwrite hash fields in place AND append a redaction-event row. Two writes against the same JSONL file:

- **Overwrite:** the original row's `prompt_hash` and `response_hash` become `"[REDACTED]"`. All other fields stay (trace_id, timestamp, caller, prompt_name, model, classification, tokens, cost). The row's `status` flips to `redacted`. New fields populate: `redacted_at`, `redaction_id`.
- **Event-row append:** a fresh entry with `prompt_name="audit.redact"`, the acting identity, and a `redaction_event` block describing target trace_id, reason, justification, and approvers.

Example (after redaction):

```
{"trace_id":"abc","caller":"automation.ingest","prompt_hash":"[REDACTED]","response_hash":"[REDACTED]","status":"redacted","redacted_at":"2026-04-25T...","redaction_id":"r-7890"}
{"trace_id":"r-7890","caller":"user@example.com","prompt_name":"audit.redact","status":"ok","redaction_event":{"target_trace_id":"abc","reason":"gdpr-art-17","justification":"data subject request DR-2024-042","approvers":["alice","bob"]}}
```

Atomicity is achieved through temp-file-and-rename; tests verify that a crash leaves either pre-redaction or post-redaction state, never partial.

**Redaction reasons (closed enum):** `gdpr-art-17`, `data-residency`, `platform-rotation`, `legal-hold-release`, `operator-error`. A non-empty `justification` is mandatory.

**RBAC + multi-approver:** a new `REDACT_AUDIT` capability assigned only to platform-admin. PR-time approval workflow requires **two platform-admin approvals** on any PR that commits a redaction event — DORA's two-person rule. A single admin can issue the local redaction immediately (statutory clock under GDPR), but cannot retain it in the committed log without a second co-sign.

**Surfaces:** CLI (`kiri audit redact --trace-id <id> --reason <enum> --justification <text>`), web UI button gated to platform-admin. The MCP tool `kirimana.audit.redact` is **explicitly out of scope** — external AI assistants must never redact our audit; that would invert the trust model.

**Explicitly NOT supported:** bulk redaction (per-trace_id only; bulk retention belongs in the SIEM); silent rotation (forbidden locally); content-sensitive substring redaction (the cryptographic guarantee was the hash); undo (defeats Article 17).

**SIEM note:** production deploys ship audit JSONL onwards to Splunk / Sentinel via standard log-shipping. Redacting locally only cleans the local copy; the SIEM has its own retention policy. The CLI help and docs spell this out.

## Consequences

- **Positive:** GDPR Article 17 / EU AI Act / DORA compliance becomes operational rather than theoretical. The redaction operation is itself fully auditable — a hostile platform-admin leaves two trails plus a PR-review requirement. No new package, format, or infrastructure; the change is additive on existing classes. The pattern generalises to other append-only logs.
- **Negative / costs:** "Redact, don't delete" is harder to explain than `purge`; an alias that errors with a pointer mitigates surprise. Two-approver gating slows operational response, but a single admin can perform the immediate local redaction within the statutory window. Local plus SIEM redaction is a two-step operation that customers may miss.
- **Neutral:** Time-locked PGP-signed redaction proofs and apply-log redaction are deliberate follow-ups, not MVP.

## Related

- See [arch-06-audit-lineage](../architecture/arch-06-audit-lineage.md)
- Related ADRs: 016 (AI Gateway trust ladder), 017 (MCP), 026 (compliance generators)
