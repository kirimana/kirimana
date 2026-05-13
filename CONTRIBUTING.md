# Contributing to Kirimana

Thank you for your interest. Here is how the project handles contributions.

## Current status: private preview

Kirimana is in private preview (v0.9). External code contributions are not yet open — the public repository contains documentation, architecture decisions, and governance only. Source code goes public alongside v1.0 (target H2 2026).

In this window, the contributions we **can** accept and value are:

- **Documentation fixes** — typos, broken links, factual errors in ADRs or architecture docs
- **Discussions** — opening issues to flag confusion, questions, or use-cases we should know about
- **Bitol / ODCS feedback** — if you're a contributor to the ODCS standard, our proposed `ai_policy` extension (see [ADR 008](docs/adr/008-ai-policy-extension-proposal.md)) is open for comment

## After v1.0 (GA): full contributions open

When the source code goes public, we welcome:

- Bug fixes and feature work via PR
- New adapters following the conformance contract ([ADR 012](docs/adr/012-adapter-conformance-versioning.md))
- New ADRs proposing architectural changes
- Pack development against the public Pack API

## Sign your commits — DCO

Kirimana uses the [Developer Certificate of Origin](https://developercertificate.org/). Every commit must include a `Signed-off-by:` line. This is a lightweight assertion that you have the right to contribute the code; it does not require a corporate CLA.

To add the sign-off automatically: `git commit -s`.

We do not use a CLA.

## Style

- Python: `black`, `ruff`, `mypy --strict` on touched modules. Configured in `pyproject.toml` (visible at GA).
- Markdown: one sentence per line for diff readability is preferred but not required.
- Commit messages: short imperative subject ("Add resilience taxonomy"), optional body explaining the why.

## Code of Conduct

Participation in this project is governed by the [Code of Conduct](CODE_OF_CONDUCT.md).
