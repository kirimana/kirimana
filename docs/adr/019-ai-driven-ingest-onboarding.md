# ADR 019 — AI-driven ingest onboarding

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** original ADR 0028 (archived)

## Context

Authoring a new ingest source today requires understanding the `sources/*.yml` schema, the chosen connector's source-definition catalog (Airbyte, dlt, Debezium, native, or landing-zone), the vault-reference convention, the classification taxonomy, and the domain policy. Each step is learnable; together they are a cliff for first-time users.

The existing `contract new` command demonstrates the shape of the answer — an AI-assisted conversational flow that asks one question at a time, proposes a valid artefact, and lets the user review and accept. Nothing equivalent exists for ingest sources. The Source Setup page in the web UI is one surface; a CLI plus AI-assisted flow plus editor skill is a cleaner onboarding path that fits git-driven teams.

## Decision

Introduce a `kiri ingest new-source --interactive` command that drives an AI-assisted conversation.

- **Backend selection is the user's first decision, never a silent default.** If `--backend` is omitted, the AI's first question asks the user to choose between Airbyte (recommended for connector coverage), native, dlt, Debezium, or landing-zone, with a one-paragraph decision matrix.
- **AI gateway routing.** The flow runs as `AIGateway.generate(prompt_name="<backend>_source_setup")`. Prompts are versioned YAML, audit-logged per turn, classification-gated. Each backend ships its own prompt so question flow matches the backend's conceptual model.
- **One question at a time.** The AI asks source type, deployment, streams or tables, classification intent, domain assignment, and sync mode. It proposes a draft `sources/<name>.yml`, the user reviews via diff, then confirms save. `contract lint` runs automatically post-save.
- **Reference skill `add-source`** ships per the skills ADR, providing a `/add-source` entry point for editor users.

Four non-negotiable security invariants, enforced in code, docs, and tests:

1. **Audit on every AI call** — same prompt-hash, response-hash, caller, classification, cost log used everywhere else.
2. **Credentials never flow through the AI surface.** The prompt instructs the model to refuse credential input and instead suggest a vault reference. A client-side tripwire blocks user messages resembling tokens or API keys.
3. **No auto-persist.** AI proposes; user explicitly confirms save.
4. **AI policy enforcement applies.** If the chosen classification is confidential or restricted, downstream AI calls in the session inherit the enforcement.

The web UI Source Setup page stays — it drives the same versioned prompt with a different caller value. Users pick whichever entry point fits their workflow.

## Consequences

- **Positive:** Time-to-first-source drops from hours to minutes. Classifications and governance happen upfront rather than as an afterthought. One prompt drives multiple surfaces (CLI, UI, future MCP tool). The `--backend` flag extends cleanly to dlt, Debezium, and custom sources without new UI code.
- **Negative / costs:** AI quality varies by source-type familiarity — common sources get strong defaults, obscure connectors get vague proposals (mitigated by the review step). Requires an AI provider credential; stub-mode works but yields generic output. Conversation state lives in memory, not git, until the final YAML lands.
- **Neutral:** The hand-authoring path is unchanged. MCP exposure (`kirimana.ingest.propose_source` as a tool) is a follow-up once MCP write tools land.

## Related

- See [arch-04-ai-gateway](../architecture/arch-04-ai-gateway.md)
- Related ADRs: 016 (AI Gateway trust ladder), 018 (skills)
