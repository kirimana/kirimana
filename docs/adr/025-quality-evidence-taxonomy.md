# ADR 025 — Quality evidence taxonomy

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** original ADR 0064 (archived)

## Context

Three ADRs together produce compliance evidence about data quality:

- Quality rules live on contracts and emit to dbt or future Great Expectations.
- The compliance report uses dbt-test pass rate as DORA data-quality evidence.
- DQX is the proposed Databricks-native engine; until then, ODCS-only expectations are documentation only.

The failure mode: a compliance report can report "DQ checks passed" when in fact no DQ engine ran. The rules existed; they just were not executed. Auditor sees green, regulator sees green, nothing actually validated the data. This ADR locks an explicit evidence-status taxonomy so reports distinguish *passed because checks ran and succeeded* from *passed because no checks ran*.

## Decision

Compliance reports include a per-contract `quality_engine.status` field sourced from the runtime state of the contract's quality run — never from the contract definition alone.

| Status | Meaning | DORA-DQ verdict |
|---|---|---|
| `engine_ran` | A quality engine (dbt-test, DQX, GE) executed the contract's expectations and recorded pass/fail | Counts toward DORA-DQ evidence |
| `documentation_only` | The contract declares expectations but no engine is configured for this target | **Inconclusive** — does NOT count as passing |
| `engine_none_configured` | No quality engine configured at the project / target level | **Inconclusive** — does NOT count as passing |
| `engine_not_applicable` | The contract has no expectations (bronze pass-through with no DQ rules) | Neutral; flagged so reviewers can decide if missing rules are intentional |

**Report-level rules:**

- The DORA-DQ section MUST show counts broken down by status (e.g. "127 contracts: 84 `engine_ran` pass, 5 `engine_ran` fail, 22 `documentation_only`, 16 `engine_not_applicable`").
- The DORA-DQ overall verdict MUST be `inconclusive` if any contract is `documentation_only` or `engine_none_configured`, unless the project explicitly opts in via `compliance.quality_engine.accept_documentation_only: true` in `dca.yml`. The opt-out is logged in the report's "deliberately accepted gaps" section.
- Engine-version metadata (dbt 1.9.4, GE 1.5.0, DQX 0.3.1) is rendered for any `engine_ran` row.
- Report consumers MUST NOT roll `documentation_only` rows into the same bucket as `engine_ran` rows.

**Per-engine emission:** dbt-test emits `engine_ran` with version plus per-test result. DQX and Great Expectations emit the same shape when wired. If nothing is configured the runtime emits `engine_none_configured` (project level) or `documentation_only` (per-contract).

Until DQX lands, only dbt-test runs. Contracts implemented as dbt generic tests get `engine_ran`; ODCS-only contracts get `documentation_only`. This is honest representation of the current state.

## Consequences

- **Positive:** Compliance evidence stops looking green when it isn't. A regulated customer running the report against a project with no DQ engine sees `inconclusive`, not `pass`. Auditor trust preserved — "we don't have the engine yet" is defensible; "we said we passed but didn't run anything" is a credibility hit. The roadmap clarifies: once a real engine lands, `engine_ran` fills the gap and verdicts flip linearly, not via a redesign.
- **Negative / costs:** More-honest reports look worse than today's. Customers running today's report on a project without a DQ engine see `inconclusive` instead of `pass`. The opt-out flag is a foot-gun if abused; the "deliberately accepted gaps" section surfaces every use.
- **Neutral:** Small per-contract metadata field; storage and serialisation cost is negligible. Existing CI gates consuming the report's overall verdict need to handle `inconclusive` — most likely by failing, which is the right behaviour for regulated deployments.

## Related

- See [arch-07-compliance](../architecture/arch-07-compliance.md)
- Related ADRs: 008 (medallion), 026 (compliance generators), 027 (quality engine path)
