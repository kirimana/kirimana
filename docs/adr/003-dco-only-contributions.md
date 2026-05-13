# ADR 003 — DCO-only contribution model (no CLA)

- **Status:** Accepted
- **Date:** 2026-05-13

## Context

Open-source projects use one of two mechanisms to make contributions legally acceptable: a Contributor License Agreement (CLA), or the Developer Certificate of Origin (DCO). A CLA is a formal agreement that assigns or licenses copyright to a designated entity, often with rights that permit later re-licensing of the codebase. The DCO, originally written by the Linux kernel project, is a one-line per-commit attestation that the contributor has the right to submit the code under the project's existing license.

A CLA carries weight only when a project might re-license. Kirimana has committed to Apache-2.0-forever (see ADR 001), which forecloses that need. The CLA's main remaining function — letting a corporate steward unilaterally re-license contributions — is the exact power ADR 001 promises never to use. Keeping a CLA in that context misrepresents the commitment.

Contribution friction is also a real cost. Every CLA adds an out-of-band signing step, often a corporate-counsel review step, sometimes weeks of delay. For first-time drive-by contributors fixing a typo, the friction is fatal. The Linux kernel and most Apache Software Foundation projects use DCO precisely because it preserves legal sufficiency at near-zero friction.

## Decision

**Kirimana accepts contributions under the Developer Certificate of Origin (DCO) 1.1, with no CLA.**

Operational rules:

- Every commit carries a `Signed-off-by:` trailer matching the committer's real name and email. The DCO bot enforces this on every pull request.
- Contributors retain copyright on their contributions. They license those contributions to the project under Apache 2.0 — the same license the rest of the codebase already uses.
- The `CONTRIBUTING.md` file in the repository root explains DCO, the sign-off mechanism (`git commit -s`), the pull-request workflow, and the development setup.
- The Contributor Covenant 2.1 is the Code of Conduct (`CODE_OF_CONDUCT.md`).
- A separate, lightweight trademark policy governs use of the "Kirimana" name in derivative projects, forks, and integrations. The trademark policy is the only legal artefact that limits what third parties may publish — it is intentionally narrow and human-readable.
- If a future commercial spinout genuinely needs a CLA (for example, a separately governed product that depends on Kirimana code under a different license), it requires its own ADR and its own legal vehicle. The core remains DCO-only.

## Consequences

- **Positive:** Minimal contribution friction — a one-line attestation, no signing ceremony, no corporate-counsel review for typo fixes. Aligns with Linux-kernel and Apache-project precedent that procurement and legal teams already recognise. Reinforces the Apache-2.0-forever commitment (a CLA without intent to re-license is theatre).
- **Negative / costs:** Contributors need brief education on DCO and `git commit -s`; mitigated through `CONTRIBUTING.md` and the bot's failure message. Without a CLA the project cannot relicense, even with universal consent, without contacting every contributor — by design.
- **Neutral:** The trademark policy is a separate document because Apache 2.0 grants copyright and patent rights but is silent on trademark. Most users will never notice.

## Related

- See [arch-01-overview](../architecture/arch-01-overview.md)
- Related ADRs: 001, 002
