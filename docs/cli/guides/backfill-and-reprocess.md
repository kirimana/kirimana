# Backfill and reprocess

Sometimes data was already applied, but wrong — a bad window, a corrected source, a schema fix that needs history rebuilt. Kirimana gives you controlled, audited ways to reprocess without silently losing history or double-counting. This guide covers watermark-window backfill for flat silver (`kiri apply`) and declared satellite rebuild for Data Vault (`kiri dv apply`), and explains the cursor-advance semantics that make both safe to re-run.

## Watermark-window backfill (`kiri apply`)

Flat watermark contracts advance a stored cursor as they process. To reprocess a bounded range, use `--window '<START>..<END>'`, optionally with `--reset`.

The mechanics turn on whether you pass `--reset`:

- **`--window` alone** — bounds this run's **upper** to `END` without rewinding. The lower bound stays the stored cursor; you are catching up *to* `END`, not reprocessing behind it.
- **`--window` with `--reset`** — uses `START` as this run's **lower** bound, so the whole `START..END` range reprocesses. This is the actual backfill. It is idempotent: the model's pre-hook deletes the window before rewriting it, so re-running produces the same result.

In both cases the **stored cursor advances to `END` only on success**. A failed run leaves the cursor unchanged — you can safely retry.

A `--window` backfill **must** build dbt: it cannot be combined with `--skip-dbt`, `--contract`, or `--domain` (all of which skip the dbt build). Advancing the cursor without materialising the model would lose data, so the apply refuses rather than silently advance.

### Worked example

Say a flat contract's cursor is at `2026-05-31` and you discover the source for the first week of May was corrected upstream. You want to reprocess `2026-05-01..2026-05-07` and then let normal runs continue.

```bash
# Reprocess exactly that week. START becomes the lower bound because of --reset.
# The pre-hook deletes 2026-05-01..2026-05-07, then rewrites it.
kiri apply --target prod --window '2026-05-01..2026-05-07' --reset
```

If this run **succeeds**, the cursor advances to `2026-05-07`. That is behind where it was (`2026-05-31`) — which is exactly what you want to fix the past, but it means the next normal run has ground to re-cover. To catch back up to a known point without rewinding history:

```bash
# Move the upper bound forward to 2026-05-31 without reprocessing behind the cursor.
kiri apply --target prod --window '2026-05-01..2026-05-31'
```

Then resume normal cursor-driven applies:

```bash
kiri apply --target prod
```

If any of these runs **fails**, the cursor stays where it was before that run — retry the same command.

`START` and `END` are validated against the contract's watermark grain, so a malformed or mis-grained range is rejected up front.

## Declared satellite rebuild (`kiri dv apply --backfill`)

For Data Vault silver, a backfill is a *declared rebuild of historical state*. Satellites and PIT from a given batch forward are force-applied (the normal idempotency skip is bypassed); hubs and links are preserved insert-only; bridges and references are recomputed.

Because this is destructive to satellite/PIT history, it is gated:

- `--backfill` requires **both** `--from-batch` (the inclusive batch id to rebuild from) and `--reason` (an audit justification).
- Without `--yes`, `--backfill` **prints the plan and exits 0** — a dry run. Nothing is rebuilt until you confirm.

### Worked example

```bash
# 1. Dry-run first: see exactly which nodes would be force-applied, preserved,
#    or recomputed. Exits 0 without changing anything.
kiri dv apply --backfill --from-batch 2026-05-01T00 --reason "upstream correction to May source"

# 2. Once the plan looks right, confirm with --yes to run the destructive rebuild.
kiri dv apply --backfill --from-batch 2026-05-01T00 --reason "upstream correction to May source" --yes
```

The audited request — who, when, why, and `from_batch` — plus each node's disposition are stamped onto the run manifest, which becomes the backfill audit-log entry. The new manifest is written under the backend's `run-manifests/` directory and the `latest.json` pointer is updated, so the **next** ordinary `kiri dv apply` picks up incrementally from the rebuilt state.

Exit codes: `0` every node applied or skipped cleanly (or a backfill dry-run); `1` one or more nodes failed; `2` setup error.

## Cursor-advance semantics in one line

Both surfaces share the same safety rule: **the cursor (or incremental pointer) advances only on a successful run, and a declared backfill never happens without an explicit confirmation.** That is what lets you retry a failed backfill without corrupting state, and audit every intentional history rewrite.

## Related reference

- [build, plan & apply](../build-plan-apply.md) — `kiri apply` and its `--window` / `--reset` / `--materialize` flags
- [silver & data vault](../silver-datavault.md) — `kiri dv apply`, `kiri dv quality`
- [platform operations](../platform-operations.md) — `kiri reload`, `kiri history`, `kiri vacuum`
- [plan, apply & debugging](plan-apply-and-debugging.md) — diagnosing why a run failed before you retry
