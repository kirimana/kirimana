# The AI trust ladder and human approvals

Kirimana is AI-first but not AI-dependent. Every AI-driven flow is framed as **Kiri** doing the work, but Kiri only ever *proposes* — the boundary between AI output and a change to your spec, code, or platform is always a human action. This guide explains the model, the query gate that stops unreviewed AI SQL, the human approval flow, and the air-gapped local provider option.

The single most important property: **the system works identically with AI turned off.** AI accelerates authoring and detection; it is architecturally forbidden from the deterministic generation pipeline. Turn Kiri off and you lose the acceleration, not the platform.

## Every AI call goes through one audited gateway

There is no direct LLM SDK use anywhere in Kirimana. All model calls flow through a single AI gateway that logs every invocation — prompt hash, response hash, provider, model id, pinned version, token count, cost, caller, and the human decision that followed. That log is the evidence base the compliance report and audit correlation build on.

## The trust ladder

Every AI operation sits on one of four levels, with explicit default behaviour:

| Level | Operation type | Example | Default behaviour |
|---|---|---|---|
| 0 | Read-only assistance | explain a plan diff | Shown to you; nothing persisted |
| 1 | Suggestion for review | draft a contract from a schema | Typed output; **you accept before any write** |
| 2 | Structured detection | drift / PII findings | Findings produced; **you decide the response** |
| 3 | Autonomous mutation | *(none — forbidden by design)* | AI never writes spec, code, or platform state |

Level 3 is structurally absent — there is no code path for it. Even high-confidence AI does not auto-apply. AI is also forbidden from the generation pipeline (`plan` / `apply` / `verify`), validation pass/fail, DDL planning, and diff computation: those must be deterministic, and a CI check fails the build if an AI import leaks into them.

## The query gate

When an AI surface (for example an analyst chat over the warehouse) wants to run SQL, the query gate classifies the statement as **allowed**, **gated**, or **forbidden** before anything executes. You can dry-run the gate on any candidate statement — this is safe to wire as a pre-commit or CI check:

```bash
# Exit 0 = allowed, 1 = gated, 2 = forbidden
kiri ai gate-test "SELECT count(*) FROM crm.customer" --catalog main --schema crm

# From a file or stdin
kiri ai gate-test ./candidate.sql
kiri ai gate-test - < candidate.sql
```

Use `--permissive` during development to demote unresolved or unparseable statements to *gated* instead of *forbidden* — never in production. Every gate decision is written to the gate-audit log, which is exactly what `kiri audit correlate` later checks the live warehouse traffic against.

Separately, Kiri can propose a deterministic, methodology-grounded build plan — an ordered sequence of `kiri` commands, each tagged with its side-effect class. This is *propose-only*: Kiri lays out the steps; a human runs the mutating ones.

```bash
kiri ai plan "build a governed sales silver layer from the CRM source"
```

## The human approval flow

When the gate decides a query needs sign-off, it hands off to the approval flow. Operators work this flow from the CLI; the same requests are visible to the web UI and MCP server as long as everyone points at the **same backend store** (`--store` / `--store-path`). Point the CLI at a different store and you simply won't see the AI surfaces' requests.

A typical operator cycle:

```bash
# See what's pending for a workspace
kiri approve list --workspace kiri:workspace:analytics

# Record an approve vote (a single reject vetoes the request)
kiri approve grant <request-id> --as compliance@example.com --rationale "reviewed, in scope"

# Issue a use-once bearer token for an APPROVED request
kiri approve issue <request-id>
```

The plaintext token is printed **exactly once** — the store keeps only its SHA-256 hash. The operator forwards it to the requester out-of-band; the requester re-presents it on resubmission:

```bash
kiri approve consume --token <token> --sql-hash sha256:<hex>
```

Consumption is use-once and bound to the request's SQL fingerprint, so a replay or a resubmission of *different* SQL is rejected. The result is a complete, reconstructable chain: gated query → recorded votes → single-use token → consumed on exactly the query that was approved.

## Air-gapped: the local provider

For air-gapped or data-residency-constrained deployments, the gateway supports a `local` provider (Ollama) as a first-class peer to the cloud provider — the AI surfaces, the gate, and the approval flow all behave the same way. Local models may produce lower-quality drafts than a cloud model; the gateway surfaces that honestly via capability declarations rather than pretending parity. No prompt or response leaves your perimeter.

And, again: with **no** provider configured at all, every non-AI workflow runs unchanged. The trust ladder, the gate, and the approval flow exist precisely so that adding AI never adds surprise autonomy.

## Related reference

- [AI assistants](../ai-assistants.md)
- [Security and compliance](security-and-compliance.md)
