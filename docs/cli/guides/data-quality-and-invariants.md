# Data quality and invariants

Kirimana treats quality as evidence, not vibes. A silver layer isn't trustworthy because someone says so — it's trustworthy because there is a machine-readable record of tests that ran and passed, business rules that hold, and referential integrity that is intact. This guide covers the four surfaces that produce that record: quality evidence, business-rule invariants, the foreign-key gate, and Data Vault referential integrity.

All four are deterministic — no AI is involved in a pass/fail decision.

## Quality evidence

`kiri quality collect-evidence` turns a `dbt` test run into per-contract evidence rows. It reads the dbt project's `run_results.json` and `manifest.json`, walks the test→model attachment graph, maps each model back to the Kirimana contract that owns it, and writes one JSON-per-line `ContractEvidence` row.

```bash
# After a dbt build/test, collect the evidence
kiri quality collect-evidence --target-dir ./target
```

By default the rows land in `<project>/.kirimana/quality_engine/evidence.jsonl`. This file is what the compliance report's data-quality section consumes — so `kiri compliance report` reflects real test runs, not assertions. Run `collect-evidence` as the step immediately after your dbt test job and your compliance posture stays current for free.

```bash
kiri quality collect-evidence -t ./target -o ./evidence.jsonl
```

## Business-rule invariants

Beyond structural tests, contracts can declare **business-rule invariants** — statements that must hold about the *meaning* of the data (a balance never negative, a total equal to the sum of its parts). `kiri invariants check` evaluates these against the active target's silver layer.

```bash
kiri invariants check --target prod

# Only the error-severity rules, machine-readable, fail the build if any warn-rule breaches
kiri invariants check --severity error --format json --strict
```

Severities are `error`, `warn`, and `drop`. By default `check` evaluates all of them and reports; `--strict` makes a warn-severity breach exit non-zero so you can gate a merge or a deploy on it.

## The foreign-key gate

`kiri fk gate` probes the foreign keys declared in your contracts against what's actually in the silver layer — catching orphaned children, dangling parents, and self-cycles before they reach a report.

```bash
kiri fk gate --target prod
```

Two checks (FK-001, FK-002) can be excluded with `--exclude` when you have a documented reason. Two cannot: FK-003 (dangling parent) and FK-004 (self-cycle) are **structural hard fails** and are never suppressible — an orphaned reference or a reference cycle is a defect by definition. The self-cycle probe runs to a bounded depth (`--max-cycle-depth`, default 16); a cycle longer than the bound is *surfaced in the finding*, never silently capped.

## Data Vault referential integrity

For Data Vault silver projects there is a dedicated quality catalogue covering hub/link/satellite integrity. Run it against the target's live silver layer:

```bash
kiri dv quality run --target prod
kiri dv quality list                 # print the catalogue (id, name, severity, scope)
kiri dv quality explain <check-id>   # description + references for one check
```

For pre-merge testing without a live warehouse, evaluate the same DV invariants against CSV fixtures — each `<table>.csv` must match a DV contract's silver-build column shape:

```bash
kiri dv test --fixture-dir ./fixtures
kiri dv test --fixture-dir ./fixtures --output json
```

Fixture-based `dv test` is the fast, deterministic check you run in CI; `dv quality run` is the live-warehouse check you run after an apply.

## A representative quality loop

```bash
kiri dv test --fixture-dir ./fixtures            # fast, pre-merge, no warehouse
kiri fk gate --target prod                       # referential integrity
kiri invariants check --severity error --strict  # business rules hold
kiri dv quality run --target prod                # DV integrity, live
kiri quality collect-evidence -t ./target        # record the evidence for compliance
```

## Related reference

- [Data quality](../data-quality.md)
- [Silver / Data Vault](../silver-datavault.md)
- [Security and compliance](security-and-compliance.md)
