# arch-06: Audit and Lineage

Audit and lineage in Kirimana are not features added on top of the contract lifecycle — they are byproducts of it. Every plan, apply, verify, federation resolution, and AI Gateway call emits an audit event in a single canonical format; every contract carries column-level lineage references that the planner resolves and the catalog persists. This document describes the JSONL audit format, the lineage model, OpenLineage export, audit redaction for compliance reads, and the trace-ID correlation pattern that joins Kirimana audit to Databricks platform audit.

## JSONL audit format

Audit is append-only JSONL. One file per project per day, written via the catalog backend with a write-ahead behaviour that survives apply failures. Each line is a self-contained event:

```json
{
  "ts": "2026-05-12T08:14:22.918Z",
  "event": "contract.apply",
  "actor": "user@example.com",
  "actor_type": "human",
  "trace_id": "01HXR8...ULID",
  "project": "sales-dv",
  "domain": "sales",
  "env": "prod",
  "contract_urn": "urn:kirimana:contract:sales.customer:1.4.0",
  "adapter": "databricks@1.4.0",
  "result": "applied",
  "plan_hash": "sha256:...",
  "on_behalf_of": null
}
```

Events come from a closed taxonomy (contract.plan, contract.apply, contract.verify, federation.resolve, ai.call, quality.run, catalog.write, rbac.decision, redaction.applied). The taxonomy is versioned alongside the contract schema.

The `trace_id` is a ULID generated at the entry point (CLI invocation, web request, MCP call) and propagated through every internal call and every adapter call. It is the join key for cross-system correlation.

## Column-level lineage

Lineage is contract metadata, not a separate graph extracted from query logs. Each contract property carries a `dca.lineage.from_columns` customProperty listing the URNs of the upstream columns it derives from:

```yaml
properties:
  - name: customer_lifetime_value
    customProperties:
      - property: dca.lineage.from_columns
        value:
          - urn:kirimana:column:sales.orders:1.2.0#net_amount
          - urn:kirimana:column:sales.customers:1.1.0#tenure_months
```

The validator (arch-02) lints two things: the reference list is non-empty for derived columns, and every URN resolves through the federation resolver. Unresolvable references are an error, not a warning. The catalog persists the resolved edges as a column-level lineage graph queryable from the web app and via the MCP `read_catalog` tool.

Because lineage is written by the contract author and validated at apply time, it is accurate by construction — there is no inference step that can drift from the SQL.

## OpenLineage export

Kirimana exports its lineage graph in [OpenLineage](https://openlineage.io) format. The export is one-way and event-shaped: every successful apply emits an OpenLineage `RunEvent` describing the job, the input datasets (from contract bindings), and the output datasets (from the contract being applied). Consumers like Marquez, DataHub, or platform-native lineage UIs ingest the events through standard OpenLineage transports.

Column-level lineage rides as a facet on the dataset event, so consumers that understand the `columnLineage` facet get full granularity; older consumers degrade gracefully to dataset-level lineage.

## Audit redaction

GDPR and equivalent regimes require the ability to redact personal data from records — including audit records — on request. Kirimana implements redaction, not deletion: a redaction event is appended to the audit stream with a reference to the original event, and a hash-overwrite is applied to the sensitive fields of the original. The chain remains continuous; the values become unrecoverable.

Redaction requires the platform-admin role plus a second approver (RBAC arch-08). Every redaction event is itself audited (`redaction.applied`). Compliance reports (arch-07) read past redacted fields without dereferencing them — the evidence is that an event occurred, not the personal detail within it.

## Databricks trace-ID correlation

On Databricks targets, the trace_id from the Kirimana audit event is injected into two places: the Databricks SQL statement comment (`/* kirimana_trace_id=01HXR8... */`) and the Workflow run's tag set. This means a Databricks platform audit log line and a Kirimana audit event can be joined on a single key, end to end.

The result: a customer running on Databricks has one queryable audit chain spanning intent (Kirimana contract apply) and execution (Databricks workflow run), without writing a correlation layer of their own. The pattern generalises — adapters for other platforms expose the same `trace_id` through whatever per-statement metadata channel they support.

## Related ADRs

023, 024, 025
