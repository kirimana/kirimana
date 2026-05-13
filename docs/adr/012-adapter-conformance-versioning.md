# ADR 012 — Adapter conformance versioning and test suite

- **Status:** Accepted
- **Date:** 2026-05-13
- **Re-curated from:** originally ADR 0067 (archived)

## Context

The two-category adapter model (see ADR 011) defined `PlatformAdapter` and `FrameworkAdapter`. Subsequent design work added an `OrchestrationAdapter` (see ADR 015) and a number of supporting plugin surfaces (catalog backends, vault providers, quality engines, AI providers, lineage emitters, RBAC providers, incident dispatchers, semantic exporters, MCP resource providers, federation resolvers).

Each surface was individually defensible. Collectively, the public extension surface grew without a unified versioning contract. Third-party adapter contributors could not tell which surfaces were stable, which methods were required, or what a deprecation cycle looked like. Internal callers could not tell which surfaces were plugin contracts (extensible) versus internal protocols (free to evolve without notice).

This ADR locks one versioning model across every adapter surface.

## Decision

**Every adapter surface in Kirimana is a typed Python protocol declared in a central protocols package, tagged with two coordinates:**

1. **Surface type:** `plugin` (third-party-extensible) or `internal-protocol` (Kirimana-internal, may change without notice).
2. **Surface version:** semver `MAJOR.MINOR.PATCH`, with explicit deprecation rules per category.

Each adapter implementation declares which surfaces it implements and at what version via a capability manifest. A conformance suite verifies that the manifest matches actual behaviour.

### Semver rules for plugin surfaces

- **MAJOR bump** when a method signature changes incompatibly, a method is removed, or a new method becomes required without a default.
- **MINOR bump** when new optional methods are added, or new optional parameters are added with defaults.
- **PATCH bump** for behaviour-only changes (performance, bug fixes, internal refactors invisible to implementors).
- **Deprecation window.** A method marked `@deprecated` in v1.5 must keep working through v1.6 and v1.7; removal allowed in v2.0.

### Internal protocols

Internal-protocol surfaces can change without notice across releases. Breaking changes appear in release notes but do not require deprecation cycles. The category is small on purpose — a surface is `plugin` unless there is a concrete reason it is internal.

### Capability manifest

Every adapter ships a `manifest.yml` declaring which surface and version it implements, which optional methods it wires, which it skips, capability flags (`ai_policy_aware`, dialect tag, etc.), and a conformance-suite-passed timestamp and version. The manifest is machine-checkable. Adapter loaders refuse to load adapters whose `surface_version` is incompatible with the current Kirimana release.

### Conformance suite

Each plugin surface ships a runnable conformance suite that verifies: every required method is implemented, return types match, documented error cases are handled the documented way, and per-method invariants hold (`health()` is read-only, `apply()` is idempotent, `query()` is side-effect-free). Third-party authors run it via `kiri adapter conformance-test --adapter <path> --surface <name>`. Passing updates the manifest timestamp. CI checks freshness against a configurable window (default 90 days).

### Retrospective versions

Existing plugin surfaces get v1.0.0 as of this ADR (modulo proposed surfaces at v0.x). Future expansion adds methods at MINOR bumps with default implementations on the protocol base; required methods only enter at MAJOR bumps.

## Consequences

- **Positive:** One versioning rule for all adapter surfaces; capability manifests are machine-checkable — `kiri adapter list` and pack validation refuse to load adapters that do not conform; conformance suites give adapter authors a confidence signal; deprecation windows are explicit; the plugin / internal-protocol distinction prevents over-extension (internal protocols stay internal until they earn plugin status).
- **Negative / costs:** Capability manifest format, conformance framework, the conformance CLI, and retrospective version-and-manifest updates across every in-repo adapter — a multi-week effort that has to be maintained as surfaces evolve; suites for proposed surfaces will be incomplete; the first MAJOR bump on any plugin surface (when `PlatformAdapter` eventually goes to v2.0) requires every adapter author to update — mitigated by long deprecation windows and clear release-note signalling.
- **Neutral:** A future multi-surface manifest format (single adapter package implementing several surfaces at once) is an easy extension and is deferred until the first multi-surface case lands.

## Related

- See [arch-03-adapters](../architecture/arch-03-adapters.md)
- Related ADRs: 005, 011, 015
