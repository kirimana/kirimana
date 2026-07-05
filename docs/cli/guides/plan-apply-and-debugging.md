# Plan, apply, and debugging

Kirimana follows a **plan → apply → verify** lifecycle. Generation is an *additive overlay*: Kiri owns a well-defined set of generated files and never touches the rest of your tree. This guide covers reading a plan, running an apply, inspecting what was generated, and working through a failed apply methodically.

## The lifecycle

1. **`kiri plan`** — compute what an apply would do and surface metadata conflicts. Read-only.
2. **`kiri apply`** — run the pipeline: ingestion → AI description → contracts → dbt build.
3. **`kiri verify`** — compare observed platform state against the planned output.

Alongside these, three introspection commands answer "what exists and where did it come from?": `kiri list-generated`, `kiri explain`, and `kiri verify-generated`.

## Plan

```bash
kiri plan --target dev
```

The plan reports, per source table, the predicted changes, plus silver and gold contract previews and a conflicts list. Conflicts are metadata inconsistencies caught before they can corrupt a build — for example, a source table asking for SCD2 history while its update strategy would discard the versions SCD2 needs.

Two flags matter in automation:

```bash
kiri plan --target dev --strict        # exit non-zero if any conflict is found
kiri plan --target dev --format json   # machine-readable summary; diff two plans
```

Use `--strict` in CI as a gate; use `--format json` when a downstream tool needs to compare plans.

## Apply

```bash
kiri apply --target dev
```

For `rest_api` and `database` sources, an implicit fetch into the landing zone runs first. Disable it to reuse already-landed files:

```bash
kiri apply --target dev --skip-fetch
```

Scope a run while iterating — this is the key to fast debugging:

```bash
kiri apply --target dev --contract customer   # one contract (atomic re-run)
kiri apply --target dev --domain sales         # one information domain
kiri apply --target dev --skip-dbt             # ingest only; skip the dbt build
```

A scoped run (`--contract` / `--domain`) skips the dbt build by default so it stays local to the targeted materialisation. When you *do* want a scoped run to actually build its models (for example an orchestrator task running one contract), add `--materialize`.

## Reading what was generated

**Enumerate the generator-owned files.** This is the first thing to check after an apply — it tells you exactly which files Kiri wrote, so you know what is safe to hand-edit (everything else) and what will be regenerated:

```bash
kiri list-generated
```

**Trace a generated line back to its contract.** When a generated model looks wrong, `kiri explain` maps an artifact back to the source contract field that produced it:

```bash
kiri explain models/silver/customer.sql
```

**Assert byte-parity.** `kiri verify-generated` re-emits the generator artifacts and checks they match the committed tree byte-for-byte — the guard that catches an out-of-date generated file that was hand-edited or left stale:

```bash
kiri verify-generated
```

**Verify against the live platform.** `kiri verify` compares observed platform state to the planned output:

```bash
kiri verify
```

## Debugging a failed apply — step by step

1. **Re-read the error in isolation.** Re-run the *scoped* apply for just the failing unit so the output is small and the feedback loop is fast:
   ```bash
   kiri apply --target dev --contract customer
   ```

2. **Rule out metadata conflicts first.** Many apply failures are declared-metadata problems that `plan` already knows about:
   ```bash
   kiri plan --target dev --strict
   ```
   Fix any conflict it reports before touching SQL.

3. **Validate the contract.** A malformed or semantically-inconsistent contract fails the build. Validate strictly:
   ```bash
   kiri contract validate contracts/sales/customer.yml --strict
   ```
   For silver, technique rules are enforced here too — for example, a `scd2_curated` contract whose sequencing key resolves to an ingest timestamp fails at this stage by design.

4. **Separate ingestion from transformation.** If you can't tell whether ingestion or dbt failed, split them:
   ```bash
   kiri apply --target dev --skip-dbt        # did ingestion succeed on its own?
   kiri ingest bronze --source customers     # re-land + register bronze only
   ```

5. **Trace the offending artifact.** Once you have a specific bad model or column, map it back to its origin:
   ```bash
   kiri explain models/silver/customer.sql
   ```

6. **Reuse landed files while you iterate.** Avoid re-fetching from a remote source on every attempt:
   ```bash
   kiri apply --target dev --contract customer --skip-fetch
   ```

7. **Confirm the fix took.** After the scoped apply is green, re-emit and verify parity, then run the technique conformance:
   ```bash
   kiri verify-generated
   kiri silver conformance --report --target dev
   ```

## Related reference

- [build, plan & apply](../build-plan-apply.md) — `kiri plan`, `kiri apply`, `kiri verify`, `kiri list-generated`, `kiri explain`, `kiri verify-generated`
- [contracts & schema](../contracts-schema.md) — `kiri contract validate`, `kiri contract show`
- [silver & data vault](../silver-datavault.md) — `kiri silver conformance`, `kiri dv conformance`
- [backfill & reprocess](backfill-and-reprocess.md) — recovering from data that was applied wrong
