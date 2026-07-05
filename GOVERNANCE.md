# Governance

This file describes how decisions are made in the **Kirimana** project. It is authoritative; PR discussions and issue threads do not override it. If this file and a specific ADR disagree, the ADR wins — open a PR to reconcile this file.

## Current phase — Phase 1: BDFL

Kirimana is currently in **Phase 1: Benevolent Dictator For Life (BDFL)**.

- **BDFL:** David Barton (governance@kirimana.io)
- **Authority model:** the BDFL makes all technical, architectural, and release decisions.
- **Why this phase:** the project has no external contributors yet. A steering committee with one nominal member is a fiction; a foundation submission requires an established project. BDFL is honest about where we are.

Phase transitions are not subjective — they are triggered by documented criteria:

| Phase | Trigger to enter | Authority model |
|---|---|---|
| 1 — BDFL (current) | Project inception | David's organization makes all decisions |
| 2 — Maintainer team | ≥ 3 sustained contributors with ≥ 6 months of merged work, each nominated by an existing maintainer | Documented [`MAINTAINERS.md`](MAINTAINERS.md); lazy consensus on PRs; BDFL retains tie-break |
| 3 — Steering committee | ≥ 5 maintainers across ≥ 2 organizations, ≥ 1 year of stable releases | Voting; the BDFL role is retired |
| 4 — Foundation (optional) | Sustained external adoption, demonstrated need for vendor-neutral home | Donation to ASF, LF AI & Data, or equivalent |

When a phase trigger is met, a new ADR is written that formalises the transition, updates `MAINTAINERS.md`, and amends this file.

## Decision-making

### Code changes (all phases)

- Every non-trivial change lands via a PR.
- CI must pass (lint, type-check, tests, conformance, license-header, DCO).
- A merge signals acceptance of the change.
- In Phase 1, the BDFL is the single approver. In later phases, the approver set is defined in `MAINTAINERS.md`.

### Architectural decisions

- Significant architectural decisions require an **ADR** in the project's design-decision record.
- ADRs are proposed as a PR. Discussion happens on the PR.
- In Phase 1, the BDFL accepts or rejects the ADR. In Phase 2+, lazy consensus applies with BDFL tie-break (Phase 2) or voting (Phase 3+).
- Accepted ADRs are never silently edited — amendments go through a new ADR or a `superseded` status.

### Releases

- Releases follow semantic versioning.
- The core package (`kiri-core`) is the anchor; satellite packages pin compatible core versions.
- Release cadence is opportunistic in Phase 1; it becomes scheduled in Phase 2+.

## Code of conduct

Participation is governed by [`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md) (Contributor Covenant 2.1). The BDFL in Phase 1, and the maintainer team in Phase 2+, are responsible for enforcement.

## Security disclosures

Security issues follow the process in [`SECURITY.md`](SECURITY.md). Do **not** open public issues for security reports.

## Contributor agreement

Contributions are accepted under the **Developer Certificate of Origin (DCO)** — a `Signed-off-by:` line on every commit is required. See [`CONTRIBUTING.md`](CONTRIBUTING.md) for details. No CLA is required.

The choice of DCO over CLA is a documented project decision. A future switch to a CLA would require a new ADR.

## Changes to this file

Amendments to this file require a PR that cites the triggering ADR. Governance-model changes without an ADR are not accepted.
