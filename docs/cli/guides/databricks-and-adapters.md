# Databricks and the adapter model

Kirimana is platform-agnostic *by architecture*: the core never speaks a platform's dialect directly. Instead it talks to a `PlatformAdapter` (where does data live, how do I run DDL) and a `FrameworkAdapter` (how do I build transformations). This guide explains that seam, shows how to wire and health-check a Databricks workspace, covers the execution identity model, and gives you an honest read on which runtimes are actually exercised today.

## The adapter model

An adapter is a versioned, conformance-tested plugin. The core issues abstract plans; the adapter renders them into platform-native SQL and executes them. This is what makes a contract portable — the same contract targets DuckDB locally and Databricks in production without a rewrite.

List the registered platform adapters with their manifest and conformance state:

```bash
kiri adapter list
```

Run the baseline conformance suite against a named adapter — this is how you verify an adapter actually honours the surface contract before you trust it:

```bash
kiri adapter conformance-test --adapter localduckdb
```

The command refuses a surface that has no shipped conformance suite rather than silently passing, so a green result means something. As additional per-surface suites land, they become available here automatically.

## Honest runtime status

The agnosticism is real at the architectural level — the adapter seams and dialect-rendered SQL are designed so any platform *could* be a target. But designed-for and exercised-today are different claims, and Kirimana is deliberate about not overstating:

- **DuckDB** — the local execution runtime and first adapter. This is where you develop, test, and run the demo.
- **Databricks (Delta)** — the exercised production runtime.

Other platforms (Snowflake, Fabric, Trino, Postgres, and MSSQL as a migration *source*) are **architectural adapter surface** — the seams are there, but they are not shipped, exercised runtimes today. New platform work sequences Delta and DuckDB first. Treat anything outside DuckDB + Databricks as surface, not a supported target, until it has a passing conformance suite.

## Wiring a Databricks workspace

A third party provisions the Databricks-on-Azure workspace and hands you a partner-produced handoff YAML. `kiri databricks setup` validates that handoff, runs read-only smoke tests, and — only on success — writes a `databricks` target block into your project's `kiri.yml`:

```bash
kiri databricks setup --handoff ./handoff.yml --target prod
```

It refuses to write on a failed smoke test unless you explicitly pass `--skip-health-check`. There are narrower escapes (`--skip-workflows`, `--skip-vault`) for when a piece of the handoff isn't ready yet — use them deliberately, not by default.

## Health-checking the wiring

`kiri databricks health` re-runs the setup smoke checks plus catalog/schema/table probes and renders a pass/fail report. It is **read-only** — no production data is written and no jobs are submitted; the temporary `_kiri_health_probe` schema and table are dropped on success. It exits `0` when all checks pass (warnings are non-blocking), `1` on any failure, `2` on misconfiguration.

```bash
kiri databricks health --target prod
kiri databricks health --target prod --format json
```

The probes map directly to the grants your execution identity needs — `USE CATALOG`, `CREATE SCHEMA`, `CREATE TABLE`, `ALTER TABLE` (MODIFY for live-mode sinks), `SELECT` on `information_schema`, job-list access, and audit-log delivery. If a grant is missing, the corresponding probe fails and names it, so this doubles as your permissions checklist. Each probe has a `--skip-*` flag for when a capability legitimately isn't in scope for a given target.

One probe worth calling out: `--skip-audit-delivery` gates a check that the workspace actually delivers its audit logs somewhere. Leave it on — cross-system audit correlation (the shared trace-id join in `kiri audit`) depends on that delivery being configured.

## Execution identity

Kirimana runs against Databricks as an **Azure service principal** — a non-human identity with exactly the grants the health probes verify. Kirimana never issues `GRANT` statements itself; the platform team owns that. Use `kiri rbac emit-grants` to produce the SQL and hand it over for review.

For **MSSQL sources** (the migration path — see [Migration from legacy](migration-from-legacy.md)), the source connection authenticates via Entra. As with every credential, the secret is never in code: connection strings carry `${vault:<id>:<key>}` tokens resolved at load time.

```yaml
# example target/source connection — no plaintext, ever
password: ${vault:databricks/prod:client_secret}
```

## Related reference

- [Platform operations](../platform-operations.md)
- [Migration from legacy](migration-from-legacy.md)
- [Security and compliance](security-and-compliance.md)
