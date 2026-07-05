# Getting the AI gateway working

Every AI call Kirimana makes — drafting a contract, describing a source,
explaining a failed run — goes through **one** place: the AI gateway
(**ADR 0005**). Nothing calls an
LLM SDK directly. That gives you a single switch for credentials, model choice,
cost, and an audit trail of every prompt and response.

This guide gets the gateway working. For *when* the AI is trusted to act versus
propose, see [the trust ladder & approvals](ai-trust-ladder-and-approvals.md).

> **Kirimana works with the AI turned off.** If no provider is configured, the
> gateway returns "AI unavailable" and every command still runs — you just
> author contracts by hand instead of having Kiri draft them. AI is an
> accelerator, never a dependency.

---

## 1. The fastest path: Anthropic

The default provider is `anthropic`. Set one environment variable and you're
live:

```bash
export ANTHROPIC_API_KEY="sk-ant-…"
```

That's it — `kiri ingest new-source --interactive`, `kiri contract new`, and the
other AI-assisted flows now have Kiri behind them. Absent this key (and with no
other provider set), the AI is simply disabled.

**Prompt caching is always on** — the gateway caches the large, stable parts of
each prompt (the contract spec, the system framing) so repeated calls are
cheaper and faster. You don't configure it; it's a platform default.

---

## 2. Choosing a provider

Select the provider with `KIRIMANA_AI_PROVIDER`; give it the matching
credentials:

| Provider | `KIRIMANA_AI_PROVIDER` | Credentials |
|---|---|---|
| **Anthropic** (default) | `anthropic` | `ANTHROPIC_API_KEY` |
| **OpenAI** | `openai` | `OPENAI_API_KEY` (+ optional `OPENAI_BASE_URL`) |
| **Azure OpenAI** | `azure` | `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_VERSION` |
| **Local / air-gapped** (Ollama) | `local` | `OLLAMA_BASE_URL` (default `http://localhost:11434`) |

```bash
# Example: OpenAI
export KIRIMANA_AI_PROVIDER=openai
export OPENAI_API_KEY="sk-…"

# Example: fully local, no data leaves the host
export KIRIMANA_AI_PROVIDER=local
export OLLAMA_BASE_URL="http://localhost:11434"
```

The `local` provider is how you run Kiri in an environment where prompts must
not leave the network — the gateway drives an Ollama endpoint instead of a
hosted API, and the same audit log applies.

---

## 3. Model selection

Each provider has sensible built-in defaults, split into three tiers the gateway
routes to by task weight. Override any of them if you want a specific model:

```bash
export KIRIMANA_AI_DEFAULT_MODEL="…"   # general drafting
export KIRIMANA_AI_HEAVY_MODEL="…"     # deep reasoning (e.g. migration translation)
export KIRIMANA_AI_LIGHT_MODEL="…"     # cheap, high-volume (e.g. column descriptions)
```

Setting `KIRIMANA_AI_PROVIDER` alone is enough — you don't have to set the three
model vars unless you want to pin exact versions.

---

## 4. The audit trail

Every call is logged: prompt, response, model, token cost, and the calling
command. By default it lands at:

```
<project>/.kiri/audit.jsonl
```

Override the location with `KIRIMANA_AUDIT_LOG_PATH`. This file is your evidence
for [compliance reporting](security-and-compliance.md) (DORA / EU AI Act /
GDPR) — every AI-touched decision is traceable to a logged call. Treat it as an
append-only record; don't hand-edit it.

---

## 5. Turning AI off deliberately

To guarantee no AI calls in a run — a locked-down CI job, a demo, a regulated
batch — set:

```bash
export KIRIMANA_AI_DISABLED=1
```

The gateway reports AI as unavailable and every command proceeds without it.
This is stronger than "no key set": it's an explicit off switch you can assert
in policy.

---

## 6. A quick sanity check

```bash
export ANTHROPIC_API_KEY="sk-ant-…"
cd my_warehouse
kiri ingest new-source --interactive     # Kiri should now converse
tail -n 1 .kiri/audit.jsonl              # one JSON line per AI call
```

If the interactive flow says AI is unavailable, the key isn't reaching the
process (check the shell's `env`); if `audit.jsonl` never grows, no AI call was
made (the command may not need one).

---

## Related reference

- [The trust ladder & approvals](ai-trust-ladder-and-approvals.md) — how much Kiri is allowed to do
- [Wiring MCP: Databricks and Kiri](mcp-databricks-and-kiri-setup.md) — expose your project to an assistant
- [Security & compliance](security-and-compliance.md) — the audit log in DORA / EU AI Act / GDPR reports
- **`packages/ai_gateway/README.md`** — provider plugins and the gateway internals
