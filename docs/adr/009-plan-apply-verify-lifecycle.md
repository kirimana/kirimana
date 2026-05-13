# ADR 009 — Plan / apply / verify operational model

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** originally ADR 0004 (archived)

## Context

The contract (see ADR 006) is the source of truth. Platform code is generated from it. Every code-generation tool that has walked this path collides with the same tension: pure generation with no edits is brittle (real data engineering needs custom transforms, performance tuning, vendor-specific workarounds), and edit-the-generated-file is corrosive (silently drifts the implementation from the spec; regeneration becomes destructive).

The right shape, consistently, is to separate generated artefacts from human artefacts, provide named extension points where humans contribute, and wrap the whole thing in a predictable lifecycle.

## Decision

**Kirimana's operational model is plan / apply / verify, modelled on the Terraform mental model that data engineers already know.**

### Lifecycle

- **`plan`** — parse the contract, validate (ODCS schema plus semantic), check adapter capabilities (see ADR 011), compute the diff between current and desired state, show the user. Read-only and idempotent.
- **`apply`** — execute the plan. Framework adapters write files; platform adapters run DDL or validation queries. Apply is the only mutating operation.
- **`verify`** — re-read the target (filesystem, platform schema) and compare to what `apply` produced. Surfaces drift, partial failures, out-of-band changes.

There is no implicit "plan and apply silently" command. Combining the two requires an explicit `--auto-approve` flag and prominent CI warnings.

### Byte-deterministic regeneration

Every apply produces byte-identical output for byte-identical input. Stable sort order, normalised whitespace, no timestamps or environment-derived values in output. The generator and adapter versions plus contract hash live in exactly one file per emission target (`_generation_meta.yml`) — the only file permitted to carry version stamps.

### Extension mechanisms (three ship, a fourth is reserved)

1. **Namespaced sibling files (default).** Generated files live in clearly demarcated locations; user files live alongside, untouched, and reference generated artefacts via the framework's native mechanisms.
2. **Pre/post hooks (when ordering matters).** The contract declares hook references the user owns; the planner threads them into the apply lifecycle and records them in the apply log.
3. **Template overrides (escape hatch).** Adapters expose Jinja templates; users can place project-scoped block-level overrides under `.kirimana/templates/<adapter>/<block>`. Overrides are versioned against the adapter; mismatches warn at plan time. Heavy override use signals a missing ODCS capability — the right response is upstream contribution, not template forking.
4. **Inline extension regions** are designed but reserved for a later release: AST-anchored, side-channel-backed, quarantine-on-failure. Seven safeguards are non-negotiable; an adapter that cannot meet all seven does not offer the mechanism.

Naive marker regions (text-anchored `BEGIN USER CODE` / `END USER CODE` parsed by regex) are forbidden unconditionally. They fail in well-documented ways and silently lose data.

### Additive overlay

The generator never deletes, renames, or modifies a file it did not author. The set of files it owns is enumerable via `kiri list-generated`. Users mix generator output with their existing dbt project, their existing Databricks repo, their existing git history with confidence.

### AI is forbidden inside the pipeline

Plan, apply, and verify make zero AI calls. AI helps upstream of generation — suggesting contract additions, drafting hook content, explaining a plan diff in natural language. Generation itself is mechanical, auditable, reproducible. The boundary is architecturally enforced (the generation packages have no import path to the AI Gateway, see ADR 016) and CI-checked.

## Consequences

- **Positive:** The spec stays the source of truth; extensibility is structured, not improvised; generated output is reviewable, diffable, reproducible; the plan/apply/verify mental model is familiar to anyone with IaC background; users adopt incrementally because namespaced output drops into existing projects without overwriting.
- **Negative / costs:** "No edits to generated files outside designated extension points" frustrates users who want a quick fix — the rule does not bend (mitigated through fast contract iteration and AI-assisted authoring); inline extension regions are designed but not yet shipped, so genuine intra-artefact customisation needs mechanisms 1-3 today; builder-absorbed trade-offs (template versioning, semantic diff, explain command, override drift detection) require real engineering investment.
- **Neutral:** Hooks live in `customProperties` until ODCS adopts a native hook concept — a candidate for early upstream contribution.

## Related

- See [arch-06-audit-lineage](../architecture/arch-06-audit-lineage.md)
- Related ADRs: 006, 010, 011, 016
