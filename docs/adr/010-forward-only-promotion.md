# ADR 010 — Forward-only environment promotion

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** originally ADR 0041 (archived)

## Context

Multi-environment CI/CD is a Kirimana capability, but earlier ADRs did not specify the promotion model: how a contract change moves from a developer's laptop into dev, test, and prod, who is allowed to push it forward, what happens when prod is on fire and dev has half-finished work, and how anyone can see at a glance what is deployed where.

Without an explicit model, three failure modes are likely:

1. **Environment-branch divergence.** Teams reach for GitFlow with separate `dev`, `test`, `prod` branches. Branches drift. Prod ends up with commits that never went through dev. Audit trail breaks.
2. **Multiple Kirimana stacks per customer.** Operators provision three Helm releases. Three Postgres instances, three audit logs, three RBAC matrices. Operational tax that buys nothing for the typical customer.
3. **Hotfix-skip-dev culture.** When prod breaks, the temptation is to apply a fix directly. The audit trail then shows a SHA in prod that never existed in dev. Forensic reconstruction becomes impossible.

Customers ask in plain words: where is what deployed, can I hotfix without breaking my feature work, can I see at a glance which contract version is in test but not yet in prod. The answers must be product features, not policies on a wiki page.

## Decision

**Kirimana's promotion model is forward-only, anchored on six commitments:**

1. **One Kirimana control plane per customer realm.** Targets named `dev`, `test`, `prod` are configuration, not separate Kirimana installations. The platform adapter calls into different platform endpoints; the control plane is one thing. Multiple stacks are reserved for hard regulatory isolation or data-sovereignty splits, not the default.
2. **Single `main` branch as the source of truth.** No `dev` / `test` / `prod` branches. Trunk plus short-lived feature and hotfix branches that merge to `main`.
3. **Three environment cursors with a strict invariant.** Each environment is a cursor pointing at a specific SHA on `main`, satisfying `dev_sha` is ancestor-or-equal of `test_sha` is ancestor-or-equal of `prod_sha`. The invariant is enforced by `kiri release apply`. An attempt to apply a SHA to test that is not already in dev is rejected before any side effect. Hotfix or feature, no exception.
4. **Release manifest in git, mirrored in Postgres.** `.kirimana/release-manifest.yml` in the customer's contracts repository captures cursor state and active contract versions per environment. CI auto-commits the manifest on every successful apply. Postgres holds the same data for real-time querying by the dashboard, the CLI, and the PR bot. Postgres is real-time; git is durable and survives Postgres loss; CI reconciles divergence with an auto-fix PR.
5. **Hotfix uses the same flow, expedited.** There is no skip-dev path. A hotfix is a branch from `main`, a PR labelled `[hotfix]`, and a normal forward-promotion. The differences are: shortened review SLA (under one hour, configurable), CI skips non-essential checks (full test suite stays; doc lint can defer), manual gates remain. The audit row is tagged `hotfix: true`.
6. **Concurrency through ODCS versioning.** Multiple versions of the same contract can coexist briefly, governed by the contract state machine (see ADR 007). Breaking changes (column type change, column drop, classification escalation) require a `--breaking-ack` flag and the configured two-approver gate (see ADR 005).

Visibility surfaces are non-negotiable: a contract-by-environment matrix on the web dashboard, `kiri release status` on the CLI, a PR comment from a Kirimana-shipped GitHub Action, and Slack / Teams notifications via the existing dispatch infrastructure.

## Consequences

- **Positive:** Audit trail unbroken from author through every environment to prod; "where is X deployed?" is a one-second query on any of four surfaces; hotfix and feature work proceed in parallel without cherry-pick gymnastics; regulator-friendly (every prod-state change has a SHA, a PR, an approver, and an audit identifier); operationally simple — one Kirimana, one Postgres, one Helm release.
- **Negative / costs:** Long-lived feature branches must rebase periodically; the dual-truth (Postgres real-time plus git manifest) requires a reconciliation job whose bugs surface as drift warnings on PRs; forward-only removes the emergency escape hatch when prod is melting — the cost is a few extra minutes against an audit trail nobody can fake; the two-approver breaking-change gate slows breaking promotions by design.
- **Neutral:** Aligned with how dbt Cloud, Snowflake Snowsight, and Databricks Unity Catalog behave today, which shortens the learning curve.

## Related

- See [arch-08-deployment](../architecture/arch-08-deployment.md)
- Related ADRs: 005, 007, 009
