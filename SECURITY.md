# Security Policy

## Reporting a vulnerability

**Do not open a public GitHub issue for security vulnerabilities.**

Email security@kirimana.dev with:

- A description of the issue
- Steps to reproduce
- The affected version(s)
- Any known mitigations

We aim to acknowledge reports within 3 business days and issue a fix or mitigation within 30 days for high-severity issues. The coordinated-disclosure window is 90 days.

## Scope

| In scope                                    | Out of scope                         |
|---------------------------------------------|--------------------------------------|
| Code in this repository                     | Customer deployments                 |
| Official docker images                      | Third-party packs (report to owner)  |
| Default configuration                       | User-customised deployments          |

## Security design principles

Kirimana follows **shift-left** security:

1. **Secrets never in code or metadata.** Use `${vault:...}` references; plaintext fails CI.
2. **Audit every AI call.** The `AIService` layer logs prompts, responses, cost, and caller identity for every LLM invocation.
3. **PII declared in contracts.** Contracts carry `kiri.pii`, `kiri.gdpr`, and `kiri.classification` fields. Governance surfaces them in the catalog.
4. **No unmotivated outbound calls.** AI-provider egress is per-workspace opt-in; default is fully disabled.
5. **Dependencies are scanned.** CI runs `pip-audit` on every commit.
6. **Least-privilege adapters.** Each platform adapter documents the minimum credentials it requires.
