# ADR 002 — Governance and stewardship

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** originally ADR 0001 (governance portion, archived)

## Context

A community-driven open-source project must decide how decisions are made and who legally stewards the codebase, the trademarks, and the release infrastructure. Premature foundation governance creates process overhead before there is a community to govern. Indefinite founder-control deters contributors who want a credible seat at the table over time. The right answer is a documented, phased evolution with explicit transition triggers — not vibes.

The stewardship question is separate from licensing (see ADR 001). Even a permanently Apache-licensed project needs a named legal entity to hold trademarks, sign releases, accept donations, and run security disclosure. That role has to start somewhere.

## Decision

**David Barton Consulting AB (DBC AB) is the initial steward of the Kirimana project.** Stewardship covers: the `kirimana` trademark, the GitHub organisation, the package-registry namespaces, the security-disclosure contact, and the release-signing keys. DBC AB acts as fiduciary — it does not own the code (the code is Apache-2.0; every contributor retains copyright on their contribution under the DCO model, see ADR 003).

Governance evolves through four phases with documented triggers:

| Phase | Trigger | Authority |
|---|---|---|
| 1 — BDFL *(current)* | Project inception | Project lead makes all decisions |
| 2 — Maintainer team | At least three sustained contributors with six months of merged work | Lazy consensus; BDFL tie-break |
| 3 — Steering committee | At least five maintainers across at least two organisations; one year of stable releases | Voting; BDFL role retired |
| 4 — Foundation | Sustained external adoption and demonstrated need for vendor neutrality | Donation to LF AI & Data, Apache Software Foundation, or equivalent |

Phase transitions are formalised in their own ADRs. The foundation phase is explicitly anticipated: when the project outgrows DBC AB's stewardship capacity, the working assumption is donation to LF AI & Data (the same foundation that stewards Bitol — see ADR 008). The alternative (ASF) is on the table if LF AI & Data is not the right fit at the time.

Operational artefacts in the repository root: `GOVERNANCE.md` (current phase, transition triggers, decision process), `MAINTAINERS.md` (current maintainer list), `CODE_OF_CONDUCT.md` (Contributor Covenant 2.1), `SECURITY.md` (90-day coordinated disclosure window with a dedicated contact).

A separate trademark policy clarifies what third parties may and may not call "Kirimana" — necessary because Apache 2.0 grants copyright and patent rights but says nothing about trademark.

## Consequences

- **Positive:** Clear single point of accountability today; explicit promotion path that does not depend on goodwill; foundation handoff is anticipated, not improvised; trademark stewardship is decoupled from any individual contributor.
- **Negative / costs:** BDFL concentrates authority in one organisation early — some contributors will prefer projects that already have community governance; DBC AB carries the legal and operational cost of stewardship until phase 4.
- **Neutral:** The foundation path is named but not committed; the right destination may be different by the time the trigger fires.

## Related

- See [arch-01-overview](../architecture/arch-01-overview.md)
- Related ADRs: 001, 003, 008
