# ADR 020 — Hub-and-spoke governance catalog

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** original ADR 0025 (archived)
- **Supersedes:** original ADR 0018 (archived)

## Context

Enterprise data platforms consistently run as **hub-and-spoke**: one central platform team owns the infrastructure (warehouse, orchestration, CI, secrets), and business domains (HR, Sales, Finance, Production) own their data products within it. The governance question:

- **Centralised** — every contract change bottlenecks on the platform team.
- **Fully decentralised** — cross-domain integrations happen silently; classifications drift.
- **Differentiated** — light-touch within a domain, explicit and reviewed across domains.

The original ADR 0018 introduced `domain` as a first-class workspace concept but treated domains as labels rather than governance boundaries. Federation works across platforms but not across domains. Contract lint had no awareness of cross-domain references. This ADR formalises the hub-and-spoke semantics by promoting the domain field from "scoping label" to "governance edge".

## Decision

Every `Source` and `CanonicalContract` carries a mandatory `domain`. Each domain declares its governance policy once at the project level:

```yaml
domains:
  hr:
    owners: ["hr-stewards@example-corp.com"]
    default_classification: confidential
    approval_policy:
      classification_raise_approvers: 2
      default_approvers: 1
      cross_domain_consumption_approvers: 2
    retention_days: 2555
  finance:
    owners: ["finance-controller@example-corp.com"]
    default_classification: confidential
    approval_policy:
      classification_raise_approvers: 3
      default_approvers: 2
      cross_domain_consumption_approvers: 3
```

Every contract gains a `publishing` flag:

- **`internal`** (default) — consumable only within the owning domain. Cross-domain references trigger a lint error.
- **`published`** — declared cross-domain interface. Other domains may depend on it, but changes invoke both sides' reviewer counts.

Two new lint rules: `cross-domain-consumption` (error if A references B across domains and B is internal) and `cross-domain-publish-required` (warns when an internal contract is being consumed externally). The `release plan` command surfaces every cross-domain boundary crossing.

The CLI gains a `--domain` filter on `apply`, `contract promote`, `lineage impact`, `release plan`, and `contract lint`. The web governance UI gains a domain selector with role-based presets (`all`, `hr`, `sales`, …) showing only published contracts from foreign domains.

CODEOWNERS integration ties contract file paths to owning teams. The contract-approval workflow requires approvals from the right team rather than any approver. A new `codeowners-missing` lint rule catches contracts without an owning team.

What does NOT change: still one git repo, one Kirimana project, one metadata tree. Domain is a scoping axis, not a project boundary. No metadata duplication. No new adapter category.

This ADR supersedes original ADR 0018 (archived), which scoped domains as labels only.

## Consequences

- **Positive:** Domain teams ship at their own cadence — HR PRs route to HR stewards via CODEOWNERS without central-platform bottleneck. Cross-domain edges become visible, lint-checked, and explicitly reviewed. Maps cleanly onto existing CODEOWNERS and branch protection. Delivers data-mesh semantics without data-mesh complexity.
- **Negative / costs:** `dca.yml` grows a domain block (mitigated — projects without domains keep working under a single default domain). Cross-domain approver counts can deadlock — keep them separate from classification-raise counts. CODEOWNERS drift can silently break approvals; the lint rule catches the common case.
- **Neutral:** Single-domain projects are unaffected. Cross-domain semantics are enforced at lint and PR-review time, not at runtime — a running apply that reads a cross-domain contract does not fail because review already happened.

## Related

- See [arch-05-catalog-federation](../architecture/arch-05-catalog-federation.md)
- Related ADRs: 009 (catalog), 021 (federation), 023 (column lineage)
- Supersedes original ADR 0018 (archived)
