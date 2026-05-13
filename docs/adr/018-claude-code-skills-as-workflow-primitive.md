# ADR 018 — Claude Code skills as workflow primitive

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** original ADR 0019 (archived)

## Context

Kirimana already exposes a CLI, an AI gateway with versioned prompts, an MCP server, and a web UI. What is missing is a composition layer that bundles those primitives into named, editor-discoverable workflows. A new team member who wants to "add a source for our CRM system" has to know which CLI command to run, which flags to pass, local conventions for owner and classification, and then hand-edit the generated YAML.

Claude Code (and similar AI-first editors) provide **Skills** — repo-committed, discoverable named procedures (`/add-source`, `/promote-contract`) that the AI can invoke on the user's behalf. Skills compose shell commands, file edits, and prompts into named workflows.

Three positions considered:

1. **Ignore skills.** Rejected — leaves the editor-native workflow gap unfilled for the dev teams Kirimana targets.
2. **Fork into a skills-only product.** Rejected — vendor coupling without owning Kirimana's core IP (the metadata model and adapters).
3. **Skills as an opt-in ecosystem layer committed in the repo.** Kirimana ships a small reference set; users add project- and org-specific skills alongside. This ADR's choice.

The format (markdown + tool invocations) is open, so the coupling risk is bounded.

## Decision

Kirimana projects gain an optional `skills/` directory at the project root. Each skill is a folder containing one `SKILL.md` following Claude Code's format. Three nesting levels coexist, all distributed via git:

| Level | Location | Author |
|---|---|---|
| Reference | `skills/` of the Kirimana repo | Kirimana maintainers |
| Project | `skills/` of a user's project | Project team |
| Org | `skills/` in a shared package or submodule | Org platform team |

The MVP reference set ships five skills: `add-source`, `new-contract`, `new-dv-hub` / `new-dv-link` / `new-dv-satellite`, `promote-contract`, `release-status`. Each skill invokes the CLI rather than re-implementing logic.

Hard invariants enforced at PR review:

- **All AI calls go through the AI gateway.** A skill must not run inference inline; that would create an audit gap.
- **Classification and owner stay mandatory** — skills cannot pre-fill sensitive defaults silently.
- **Secrets never appear in skill text** — skills operate on metadata, with vault references for any credential.
- **Skill-originated CLI calls leave the same audit trail** as human invocations.

Skills are a composition layer, never a replacement for the CLI, MCP, or AI gateway. Every skill bottoms out at one of those surfaces.

## Consequences

- **Positive:** Editor-native workflows close the onboarding UX gap. Org-specific policies codify in git rather than in Confluence. Reference skills double as executable documentation. Audit invariants stay intact because skills route through the CLI.
- **Negative / costs:** Coupling to Claude Code's `SKILL.md` convention — other editors may or may not adopt it. Reference skills become surface area the maintainers own. Governance integrity depends on PR-review discipline; a future `skills lint` would close the loophole.
- **Neutral:** Skills are opt-in; non-skill users lose nothing. The format is plain markdown and can be read by humans or other tools.

## Related

- See [arch-04-ai-gateway](../architecture/arch-04-ai-gateway.md)
- Related ADRs: 016 (AI Gateway trust ladder), 017 (MCP), 019 (AI-driven ingest onboarding)
