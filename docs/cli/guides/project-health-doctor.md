# Project health with `kiri doctor`

`kiri doctor` is a one-command project-health punch-list. It runs the checks
Kirimana already ships — contract validation, silver-technique consistency,
source definitions, fixture presence, layer-contract validity, and (with a
target) live adapter health — and merges every finding into one ranked,
fix-hinted list. It's the fastest way to answer *"is my project healthy, and
what do I fix first?"* whether you're onboarding, returning to a project after a
while, or gating a pull request in CI.

## Run it

```bash
kiri doctor                 # offline checks only (no warehouse needed)
kiri doctor --target dev    # also probe the live target (adapter health)
```

The table output ranks errors first, then warnings, and shows a copy-pasteable
fix per finding:

```
                                  kiri doctor
  check      where                      problem                         fix
  ✗ contracts  contracts/sales/orders.yml  missing required owner          kiri contract validate contracts/sales/orders.yml --strict
  ! fixtures   sources/crm:customers       fixture_file has no CSV       create data/raw/crm/customers.csv
2 error(s), 1 warning(s).
skipped: adapter-health (needs a --target (no target given))
```

## What it checks

| check id | what it verifies | needs a target? |
|---|---|---|
| `contracts` | ODCS structural + semantic rules (owner, classification, …) | no |
| `technique` | each contract agrees with the project-wide silver technique | no |
| `layer-contracts` | layer-contract validity + orchestration closure | no |
| `sources` | `sources/*.yml` / `*.yaml` parse and validate | no |
| `fixtures` | every `fixture_file` table has its CSV (or `apply` will fail) | no |
| `adapter-health` | the target's adapter can be reached with its credentials | **yes** |

Checks that need a live target are **skipped** (not failed) when you don't pass
`--target` — doctor fails *open* on absent connectivity and *closed* on real
findings. Per-target ingestion overrides are honoured exactly as `apply` sees
them, so a `fixture_file` base that a target overrides to a database pull won't
raise a false fixture warning.

## Filter the run

The check ids are a closed menu — `--only` and `--skip` validate against it and
tell you the valid names on a typo:

```bash
kiri doctor --only contracts,technique     # run just these
kiri doctor --skip fixtures                 # everything except fixtures
```

## Use it in CI

`--format json` emits the full report (deterministic — safe to diff), and the
exit code makes it a single project-health gate:

```bash
kiri doctor --format json --strict
```

- exit `0` — no error-severity findings (and, without `--strict`, warnings are
  allowed);
- exit `2` — at least one error, or (with `--strict`) any warning.

`--strict` promotes warnings to failures, so a PR can't merge with an unmarked
PII column or a missing fixture.

## The same check, everywhere

`kiri doctor` and the web app's health endpoint call the *same* underlying
function, so the CLI, CI, and UI never disagree about whether a project is
healthy. Doctor itself is deterministic and needs no AI or network for the
offline checks — it runs in CI and air-gapped environments unchanged.

## Related reference

- [getting started walkthrough](getting-started-walkthrough.md) — where `kiri doctor` fits in the first build
- [contracts & schema](../contracts-schema.md) — `kiri contract validate`, `kiri contract lint`
- [targets, profiles & secrets](targets-profiles-and-secrets.md) — configure the target `--target` probes
- [plan, apply & debugging](plan-apply-and-debugging.md) — the build lifecycle doctor guards
