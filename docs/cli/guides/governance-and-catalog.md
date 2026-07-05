# Governance & the catalog

**Audience:** information architects and data stewards. **Read this** to make data findable
and governed — populating and searching the catalog, moving attributes through a review-state
machine, stamping classification, and deriving ownership from the project's declared domains.

## The catalog: structure + findability

The catalog is a governance-first index of every asset — sources, bronze / silver / gold
tables, views, dashboards, jobs — with classification, certification, PII flags, columns,
and lineage. It is populated from the project's own YAML (or an external stream), so the
catalog never drifts from the contracts:

```bash
# Create an empty catalog, then populate from project YAML
kiri catalog init
kiri catalog import

# Populate from an external NDJSON stream (e.g. piped from `kiri catalog pull`)
kiri catalog import --from-ndjson -
```

Find and inspect assets:

```bash
# List with governance filters
kiri catalog list --type gold --domain sales
kiri catalog list --classification confidential --pii
kiri catalog list --certification certified

# Case-insensitive substring search over name / qualified name / description
kiri catalog search customer

# Full detail for one asset (governance + columns + lineage)
kiri catalog show kiri:asset:silver:customer

# Immediate upstream + downstream neighbours of a URN
kiri catalog lineage kiri:asset:silver:customer
```

Findability is the point: a steward should reach any governed dataset by name, domain,
classification, or PII status in one command.

## The attribute review-state machine

Every catalog column moves through a governed review lifecycle rather than being edited
freely. The states are `new → in_progress → submitted → approved` (with `rejected` and a
re-open path). Transitions mirror a PR: submit is *open*, approve is *merge*, reject is
*change-requested*.

```bash
# Reference a column as <asset_urn>#<column_name>
kiri attr list --state submitted
kiri attr show 'kiri:asset:silver:customer#email'

# Editing flips new → in_progress on the first edit
kiri attr edit  'kiri:asset:silver:customer#email' --help

# Move through the lifecycle
kiri attr submit  'kiri:asset:silver:customer#email'
kiri attr approve 'kiri:asset:silver:customer#email'
kiri attr reject  'kiri:asset:silver:customer#email' --reason "PII category unconfirmed"
kiri attr reopen  'kiri:asset:silver:customer#email'

# Forensic history
kiri attr history 'kiri:asset:silver:customer#email'
```

`--reason` is **mandatory** on reject for forensic clarity. `kiri attr list` filters by
`--state`, `--domain`, or `--source` — the last is handy for triaging "what did the last CRM
sync discover?". Ask Kiri to propose a PII classification on a single column with
`kiri attr detect-pii`.

## Classification

Classification (`kiri.classification`: `public` / `internal` / `confidential` /
`restricted`) is mandatory on every contract and on every column whose sensitivity differs.
Rather than hand-editing hundreds of columns, stamp classifications from proposals in one
pass:

```bash
# Preview what would change
kiri contract merge-classification \
  --contracts contracts/ \
  --proposals proposals/ \
  --dry-run

# Apply, letting operator overrides win every precedence contest
kiri contract merge-classification \
  --contracts contracts/ \
  --proposals proposals/ \
  --operator-overrides overrides.json \
  --precedence operator
```

Proposals come from `kiri inventory pre-classify` (heuristics) and, optionally,
`--ai-results` (an AI-batch JSONL merged on top). `--precedence` sets the highest source
allowed through: `operator` > `ai_gateway` > `heuristic_high` > `heuristic_default`, so a
steward's decision is never overwritten by a heuristic.

## Ownership & stewardship

Ownership is *declared*, not maintained by hand. Domain owners live in `kiri.yml`; stewards
and governance policy live on contracts (`kiri.governance.stewards`, `approval_policy`,
`review_cadence_days`). Derive the repository's CODEOWNERS from the declared domain owners so
code review routes to the accountable team automatically:

```bash
# Emit a CODEOWNERS file from kiri.yml's domain owners
kiri contract codeowners -o .github/CODEOWNERS

# In CI: fail if CODEOWNERS diverges from the synthesised body
kiri contract codeowners -o .github/CODEOWNERS --check
```

The `--check` mode fences against manual edits that would bypass the declared ownership — the
governance model stays the single source of truth.

## A typical stewardship loop

1. `kiri catalog init` + `kiri catalog import` — build/refresh the index.
2. `kiri attr list --state submitted` — triage the review queue.
3. `kiri attr approve` / `kiri attr reject --reason ...` — govern each attribute.
4. `kiri contract merge-classification` — stamp classification from proposals.
5. `kiri contract codeowners --check` in CI — keep ownership honest.

## Related reference

- [Governance & catalog access](../governance-catalog-access.md) — the generated `kiri catalog` / `kiri attr` / `kiri contract` reference
- [The medallion + contracts model](./medallion-and-contracts-model.md) — where classification and ownership live
- [Column & goal lineage](./lineage-column-and-goal.md) — the catalog's lineage neighbours
