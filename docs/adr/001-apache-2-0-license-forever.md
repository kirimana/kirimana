# ADR 001 — Apache-2.0 license, forever

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** originally ADR 0001 + 0060 (archived)

## Context

A data-platform project's license choice is sticky. Changing it later requires consent from every contributor and risks a trust crisis. Multiple precedents in adjacent open-source data companies — MongoDB to SSPL (2018), Elastic to SSPL + Elastic Licence (2021), HashiCorp to BSL (2023) — show what happens when a vendor pivots from OSI-approved licensing to a "source-available" model: community trust collapses, hyperscaler forks emerge, and the long-term cost of the switch outweighs the short-term commercial gain.

Kirimana starts before revenue, before mass adoption, and before any acquired dependence on license flexibility. The cheapest moment to commit forever is the moment before there is anything to compromise. The decision also underwrites the standards-authorship strategy (see ADR 008): Bitol and the Linux Foundation accept proposals from credible open implementers, not from dual-licensed vendors.

Alternatives considered and rejected: open-core with a closed "enterprise edition" (Confluent style); BSL with delayed Apache conversion (Sentry style); SSPL to block hyperscaler hosting (MongoDB style). Each was rejected on the same ground — they trade compounding moat for near-term feature paywalls.

## Decision

**The Kirimana core ships under Apache License 2.0, forever.** "Forever" is a binding commitment with operational consequences:

- **Patent grant.** Apache 2.0 §3 protects users and contributors from patent litigation by other contributors — essential for AI-augmented tooling that touches novel techniques.
- **What "core" means.** Every package, every adapter, every service, every CLI, every AI prompt, every ADR. No "core stays Apache, enterprise edition is BSL" gymnastics.
- **What is forbidden.** Closed-source adapters; closed-source compliance generators; paid feature flags inside the CLI or core libraries; re-licensing to BSL / SSPL / AGPL / Elastic Licence or any other source-available-but-not-OSI license; CLAs that permit unilateral re-licensing.
- **Commercial value lives in layers next to the core.** Managed Kirimana, Compliance Pack subscriptions, pack-marketplace revenue share, advisory, training, hosted developer surfaces — all paid services built around an open product.
- **Reversal cost.** Any future re-license requires a superseding ADR, six months public notice, a clean fork-and-rename so the Apache-licensed codebase stays live under its current name, and a signed explanation of why the original commitment failed.

## Consequences

- **Positive:** Standards-authorship credibility; hyperscaler-fork risk neutralised (they can host us; that does not threaten us); senior-engineer hiring leverage; regulated-industry procurement trust; architectural discipline (every commercial idea must be a layer-next-to, not a feature-gate).
- **Negative / costs:** No tactical license-switch escape hatch if a hyperscaler undercuts the managed offering; open-core-with-enterprise-features revenue model is permanently off the table; slower near-term revenue ramp than gated peers.
- **Neutral:** Compliance Pack and managed Kirimana become load-bearing — there is no license-flexibility fallback if both fail.

## Related

- See [arch-01-overview](../architecture/arch-01-overview.md)
- Related ADRs: 002, 003, 004, 008
