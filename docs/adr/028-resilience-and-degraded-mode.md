# ADR 028 — Resilience and degraded-mode operation across plugin surfaces

- **Status:** Accepted (first production slice landed)
- **Date:** 2026-05-13
- **Re-curated from:** original ADR 0071 (archived)

## Context

Kirimana exposes many plugin surfaces — `PlatformAdapter`, `FrameworkAdapter`, `OrchestrationAdapter`, `CatalogBackend`, `VaultProvider`, `AIService`, `IncidentDispatcher`, `SemanticExporter`, `FederationResolver` — each integrating with an external system that can fail partially or transiently.

Without a uniform resilience contract, failure handling drifts: some surfaces swallow exceptions and return empty results (silent-success bug), some leak vendor exception hierarchies to callers, none expose a uniform health probe. Operators wanting to ask "is this adapter reachable?" must call a representative method and catch.

The `FederationResolver` proved a clean pattern: a `HealthReport` value type with three states (`OK` / `STALE` / `UNAVAILABLE`), explicit degraded-mode where a `STALE` surface still serves cached data, and a fail-CLOSED rule for security-sensitive callers. This ADR canonicalises that pattern as the contract every plugin surface satisfies.

## Decision

Every plugin surface implements a `Resilient` Protocol:

```python
@runtime_checkable
class Resilient(Protocol):
    def health(self) -> HealthReport: ...
    def degraded_mode_capabilities(self) -> set[str]: ...
```

`HealthReport` is a typed value with:
- `status`: `OK` | `STALE` | `UNAVAILABLE`
- `last_success_at`, `last_failure_at`, `failure_reason` (typed enum, never raw exception)
- `served_from`: `live` | `cache` | `none`

### Three rules

1. **Fail-CLOSED for security-sensitive paths.** Surfaces participating in policy enforcement, secret resolution, or audit emission must call `assert_fresh_for_security()` and refuse to serve `STALE` reads. Better to fail than authorise based on stale state.
2. **Fail-OPEN with degradation for read-only paths.** Catalog listing, lineage queries, federation resolution may serve cached results in `STALE` mode, returning the `HealthReport` so callers can decide whether to trust the data.
3. **No silent empties.** A surface that can't reach its backend returns `UNAVAILABLE` explicitly. Callers handle the failure; the surface never pretends success.

A typed `failure_reason` enum (`network_timeout`, `auth_failed`, `credential_expired`, `quota_exhausted`, `upstream_500`, `schema_mismatch`, `unknown`) lets credential-rotation and incident-dispatch react without parsing strings.

## Consequences

- **Positive:** Operators get one mental model. Credential rotation (one classifier across surfaces) and incident dispatch (typed failure reasons) become cleaner. Federation's degraded-mode behaviour is the template, not the exception.
- **Negative:** Every existing plugin surface needs a `health()` method and `failure_reason` mapping — incremental migration, not a single big change.
- **Neutral:** External integrators writing custom adapters get clear failure-handling guidance from the Protocol itself.

## Related

- See [arch-03-adapters.md](../architecture/arch-03-adapters.md), [arch-05-catalog-federation.md](../architecture/arch-05-catalog-federation.md)
- Related ADRs: 011 (two-category adapter model), 012 (conformance versioning), 021 (federation)
