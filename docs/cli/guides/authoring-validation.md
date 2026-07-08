# Validating your YAML

The commands that check `kiri.yml` and the layer manifests, and a map from common
error messages to their fix. Run these after any edit; they are read-only and need
no cloud connection (except the live adapter checks noted below).

## The commands

| Command | What it checks |
|---|---|
| `kiri project validate --project .` | `kiri.yml` is well-formed and internally consistent. Emits `{"ok": true, "issues": []}` when clean. |
| `kiri layer validate --project .` | All layer manifests parse and reconcile **project-wide** (cross-layer, cross-target) — catches twin-schema collisions, catalog-pattern token errors, target mismatches. |
| `kiri contract validate <file> --strict` | One manifest in depth, with strict gates on (PII→GDPR, etc.). |
| `kiri doctor --project .` | Runs **every** health check and prints a ranked, fix-hinted punch-list. `--format json` for machines; `--strict` treats warnings as failures (CI gate). Exit code 2 on failure. |
| `kiri plan --project .` | Shows the silver/gold contracts that would be generated — a fast way to confirm a new table is picked up. |
| `kiri contract schema-export` | Emits the JSON Schema for ODCS + `kiri.*`, for editor/pre-commit validation of your YAML. |

A clean `sales` example produces `kiri doctor` exit code 0 with zero findings — a
good sanity target for your own project.

## Error → cause → fix

| Message (excerpt) | Cause | Fix |
|---|---|---|
| `localized twin-schema collision … (ADR 0133 §4.2)` | A table sets `kiri.target_schema: <schema>_<locale>` while localization also renders that same twin schema. | Move the hand-authored table to a distinct schema, or drop the auto-twin and model localized surfaces as explicit entities. See the [localized-twin task](authoring-tasks.md). |
| `Columns declare kiri.pii but kiri.gdpr.lawful_basis and kiri.gdpr.purpose are required` | A PII-marked column's table/layer lacks a GDPR basis. | Add `kiri.gdpr.lawful_basis` + `kiri.gdpr.purpose` at the table or layer level. See [mark PII](authoring-tasks.md). |
| `kind=fact requires grain` / `requires at least one measure` | A gold fact is missing `kiri.gold.grain` or `kiri.gold.measures`. | Add the grain columns and at least one measure; use `kind=factless_fact` for coverage-only tables. |
| `unknown {token} in catalog_pattern` | `naming.catalog_pattern` uses a token outside `{env}`/`{domain}`/`{layer}`. | Use only the three closed tokens; literal text is fine between them. |
| `default_target=… not in targets` | `default_target` names a target that isn't declared. | Add the target under `targets:` or fix the name. |
| `technique=data_vault … execution_backend` mismatch | `silver.data_vault.execution_backend` set while `technique` isn't `data_vault`. | Only set the `data_vault` block when `technique: data_vault`. |
| naming-lock / `locked_at` error | The silver technique or naming was locked by the first contract and a later edit conflicts. | Keep the locked value, or follow the deliberate [switch-technique task](authoring-tasks.md). |

## Live checks (need a target)

Passing `--target <name>` to `kiri doctor` enables adapter-health checks against
the live platform; `kiri databricks health --target <name>` checks Databricks
wiring specifically. These need credentials in the environment and are the only
checks that reach the platform — everything above is static.

## Version note

This guide tracks the current product schema (`kiri` 1.0.0b2). Older installations
may not enforce every gate here (for example, stricter PII→GDPR enforcement is
newer). Check your build with `kiri --version`; if a field or check in this guide
isn't recognised, your CLI predates it.

## See also

- [Authoring overview](authoring-yaml-overview.md)
- [Editing `kiri.yml`](authoring-kiri-yml.md)
- [Editing layer manifests](authoring-layer-manifests.md)
- [Authoring tasks](authoring-tasks.md)
- [Project health with `kiri doctor`](project-health-doctor.md)
