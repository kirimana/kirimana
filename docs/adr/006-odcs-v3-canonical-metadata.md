# ADR 006 — ODCS v3 as the canonical metadata model

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** originally ADR 0002 (archived)

## Context

Kirimana's most foundational data structure is what a data contract is. The contract format shapes the registry schema, the validator's input, the code generator's source, the UI's editing surface, and every adapter's translation target. A bad choice here metastasises through every downstream component.

Two alternatives were considered and rejected. Inventing a Kirimana-specific contract format would let the team optimise the schema for AI consumption but would isolate the project from every existing tool that already speaks data-contract dialects, and would put the team in the business of maintaining a spec on top of the platform. Extending an existing spec with parallel Kirimana fields (a Kirimana-namespaced shadow schema) would buy local convenience at the cost of upstream alignment and would make every Kirimana extension look like a vendor land-grab to the standards community.

The discipline taken is the harder one: adopt an open standard straight, never fork, and use the standard's own extension mechanism for everything Kirimana-specific.

## Decision

**Adopt the Open Data Contract Standard (ODCS) v3** — maintained by Bitol under the Linux Foundation AI & Data umbrella — as Kirimana's canonical metadata model.

- **Version pinning.** Kirimana pins to the latest stable ODCS minor version per release. Major upgrades (v3 to v4) trigger a dedicated migration ADR.
- **Validation.** Bitol's JSON Schema is the source of truth for structural validation. Kirimana layers its own semantic validation on top for cross-field rules ODCS does not encode. Both run at plan time and in CI.
- **Extension discipline.** Kirimana-specific needs are expressed via ODCS `customProperties` under documented namespaces. Internal aliases or parallel fields outside `customProperties` are forbidden.
- **Canonical in-memory model.** ODCS YAML is the on-disk wire format. The in-memory shape is an immutable Pydantic v2 `CanonicalContract` in the core package. Every other component — adapters, generators, AI providers, UI — consumes the canonical model and never parses YAML itself.

The Kirimana extension namespaces (`classification`, `pii`, `gdpr`, `lifecycle`, `orchestration`, `transformation`, `lineage`, `observability`, `ai_policy`, `governance`, `wire_format`, `dedup`, `materialization`, `historize`, `partitioning`, `cost`) live in `docs/contracts/metadata-master.md` and are reviewed quarterly against the upstream roadmap. When a namespace proves broadly useful, Kirimana opens a pull request against Bitol to hoist it into the native spec. The local namespace set is intended to shrink over time, not grow. The `ai_policy` extension is the first such proposal (see ADR 008).

## Consequences

- **Positive:** Day-one interoperability with every tool that speaks ODCS; engineering effort focuses on automation and AI rather than spec bikeshedding; vendor-neutral foundation-stewarded format is a strong procurement signal; new contributors familiar with ODCS are productive immediately; upstream community contributes spec evolution for free.
- **Negative / costs:** Kirimana follows Bitol's roadmap cadence — features the team wants may take time to land upstream; some Kirimana needs lack natural ODCS expressions today (AI metadata, orchestration, lineage) and `customProperties` is a safety valve, not a long-term home; ODCS v3 to v4 eventually requires migration work; the team gives up the ability to design "the perfect spec" — which is exactly the point.
- **Neutral:** YAML invites whitespace bugs; mitigated by strong CI validation and editor tooling. Git is the source of truth on disk; a queryable registry sits on top.

## Related

- See [arch-02-contracts](../architecture/arch-02-contracts.md)
- Related ADRs: 007, 008, 011
