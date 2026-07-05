# Security and compliance — an end-to-end walkthrough

This guide is written for the person who has to answer *"can you prove this platform is secure and compliant?"* — a CISO, DPO, or platform security lead. It threads together the CLI surfaces that together give you: no plaintext secrets, role-based authorship control, PII and classification coverage, a tamper-evident audit trail, and machine-readable compliance evidence.

Everything here is code-first. Nothing is a dashboard toggle that can quietly drift — each control is a `kiri` command that runs the same way locally, in CI, and in production.

## 1. Secrets never live in code or YAML

Kirimana forbids plaintext secrets in project files. Configuration references a secret indirectly with a `${vault:<id>:<key>}` token; the real value is resolved at load time from an environment-backed vault (or an Azure/AWS/GCP resolver in production).

Store a secret:

```bash
# Prompts for the value; writes to <project>/.env under the vault naming convention
kiri vault set crm/salesforce --key password

# In CI, read the value from an env var instead of prompting
kiri vault set crm/salesforce --key password --from-env SALESFORCE_PASSWORD
```

Reference it in a source or contract YAML:

```yaml
connection:
  password: ${vault:crm/salesforce:password}
```

Take the safe inventory view — key names only, values redacted:

```bash
kiri vault list
```

This is the "what secrets do I have wired?" report you can hand to an auditor without leaking anything. CI fails on any detected plaintext secret, so the vault reference is the only path that survives review.

## 2. RBAC — who is allowed to change what

Authorship of governance-sensitive files is gated at PR time, not left to convention. The gate is environment-scoped and delegates identity to your OIDC provider.

Gate a pull request against the project's authorship rules:

```bash
git diff --name-only origin/main... > changed.txt
kiri rbac gate-pr \
  --submitter alice@example.com \
  --groups "acme/data-platform,acme/compliance" \
  --changed-files-from changed.txt
```

The gate exits `0` when all rules pass (and degrades cleanly to pass in a single-user project with no RBAC configured), `3` on a rule violation, and `2` on a usage error. Wire it as a required check so a PR that raises a column to `restricted` can only merge with the right authorship.

Kirimana never executes `GRANT` statements itself — issuing grants is the platform team's prerogative. Instead it emits the SQL for you to review and run:

```bash
# Read-only: prints per-role, per-zone GRANT statements for the resolved target
kiri rbac emit-grants --role analyst --zone all
```

Runtime access policy for the catalog (PII masking + classification-based access) is managed separately and idempotently:

```bash
kiri catalog policy list
kiri catalog policy upsert   # insert or update a policy, idempotent by id
kiri catalog policy show <id>
```

## 3. PII and classification coverage

Every contract and every schema property carries an `owner` and a `kiri.classification` — this is enforced by the canonical-model validator, so an unclassified column cannot ship. The CLI helps you get to full coverage and keep it honest.

Ask Kiri to classify PII on a single column. This is **suggestion-only by default** — it prints a recommendation (is_pii, category, classification, masking policy, reason) and writes nothing:

```bash
kiri attr detect-pii "kiri:catalog:crm.customer#email"

# Apply the suggestion (flips the column to in_progress so the review trail stays honest)
kiri attr detect-pii "kiri:catalog:crm.customer#email" --apply
```

Get the whole-project PII picture — every asset containing PII, with its columns and categories:

```bash
kiri catalog pii-scan
```

Stamp classifications back onto contracts in bulk, with an explicit precedence order (operator overrides win over AI, which wins over heuristics):

```bash
kiri contract merge-classification \
  --contracts contracts/ \
  --proposals .kirimana/inventory/proposals/ \
  --precedence operator \
  --dry-run          # report what would change before writing
```

## 4. Audit logging and cross-system correlation

Every AI call is audit-logged by construction (see [AI trust ladder and approvals](ai-trust-ladder-and-approvals.md)). Beyond AI, Kirimana correlates its own actions with the platform's audit log using a **shared trace id**, so an auditor can answer *"which Kirimana action produced this SQL?"* without time-window guesswork.

Join the Kirimana audit + incidents logs with the platform-delivered audit log into one chronological timeline:

```bash
kiri audit join --since 1d
kiri audit join --trace-id <trace> --format json
```

Run a post-hoc correlation against the warehouse's access audit to catch any AI query that ran without a matching gate decision:

```bash
kiri audit correlate \
  --since 2026-06-01T00:00:00+00:00 \
  --until 2026-06-02T00:00:00+00:00
```

Findings classify each query as clean, a bypass (AI query with no gate decision), a stale gate, or a forbidden query that ran. With `--dispatch` and an `incidents:` block configured, findings above a severity floor route straight to your ITSM.

## 5. Redaction, not deletion

For data-subject requests and residency obligations, Kirimana **redacts** log rows in place and appends a redaction-event row recording who, why, and when — it does not delete history, which keeps the audit trail intact for DORA and the EU AI Act while satisfying GDPR Article 17.

```bash
kiri audit redact \
  --trace-id <row-id> \
  --reason gdpr-art-17 \
  --justification "data subject request DR-2026-042"
```

The reason is a closed enum (`gdpr-art-17`, `data-residency`, `platform-rotation`, `legal-hold-release`, `operator-error`). Redaction is idempotent under retry — a second redact of an already-redacted row raises a clear error rather than double-writing. Note that this redacts the **local** logs only; if records are replicated to a SIEM, replicate the redaction request there too.

## 6. Compliance reporting

Generate a machine-readable evidence report for a framework and time window. Output goes to stdout (JSON by default, for GRC import) so it pipes cleanly; errors go to stderr so the primary stream stays parseable.

```bash
# DORA, GDPR, or EU AI Act
kiri compliance report --framework dora --period 2026-01-01..2026-06-30

# Markdown for humans
kiri compliance report -f gdpr --format markdown

# Gate merges in CI on any failing control
kiri compliance report -f ai_act --fail-on-fail
```

The report's data-quality section consumes the evidence produced by `kiri quality collect-evidence` (see [Data quality and invariants](data-quality-and-invariants.md)), so your compliance posture is backed by real test runs, not assertions.

## Putting it together

A representative security-and-compliance loop:

```bash
kiri vault set crm/salesforce --key password       # secrets out of code
kiri catalog pii-scan                              # find the PII
kiri attr detect-pii <ref> --apply                 # classify it (with review)
kiri rbac gate-pr --submitter … --changed-files-from changed.txt   # gate authorship
kiri audit correlate --since … --until …           # verify no ungated AI queries
kiri compliance report -f dora --format markdown   # produce the evidence
```

## Related reference

- [Governance, catalog, and access](../governance-catalog-access.md)
- [Secrets](../secrets.md)
- [AI trust ladder and approvals](ai-trust-ladder-and-approvals.md)
- [Data quality and invariants](data-quality-and-invariants.md)
