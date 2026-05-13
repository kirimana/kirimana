# ADR 029 — Rate-limit state in a persistent store

- **Status:** Implemented
- **Date:** 2026-05-13
- **Re-curated from:** original ADR 0042 (archived)

## Context

Per-API-key rate-limiting on the incidents-API was initially implemented as an in-memory token-bucket guarded by a thread lock. The original docstring was honest about the limitation: state lives in a Python dict that doesn't survive process restart.

Two things made that limitation untenable for production:

1. **Multi-replica Kubernetes deployments are the norm.** The Helm chart defaults to two replicas. With an in-memory limiter, the same caller's requests round-robin across replicas, each seeing half the traffic — a 60 req/min global limit becomes a *de facto* 120 req/min limit, and operators have no way to detect or compensate.
2. **Rolling restarts vanish bucket state.** Every chart upgrade resets every caller to a full bucket on at least one replica, allowing burst abuse during deploys.

The state must live somewhere that survives both replicas and restarts. Three candidates were considered: Redis (operationally heavy, one more dependency to monitor and secure), in-memory with sticky sessions (defeats the point of multi-replica), and Postgres (already required by the platform for release tracking and audit).

## Decision

Rate-limit state lives in **Postgres**, behind a `RateLimitStore` Protocol with two implementations:

- `InMemoryRateLimitStore` — default for single-replica development and tests. Same semantics as the original implementation.
- `PostgresRateLimitStore` — production implementation. Uses a `rate_limit_buckets` table with `(api_key_hash, bucket_id)` primary key, `tokens` column, `last_refill_at` timestamp, refilled lazily on read.

The store is selected via `DCA_RATE_LIMIT_STORE=memory|postgres`, defaulting to `memory` for local development and explicitly set to `postgres` by the multi-replica Helm overlay.

A `psycopg_pool` keeps connection cost bounded. Reads use `SELECT ... FOR UPDATE` with a short transaction window; the bucket arithmetic is a single round-trip. Benchmarks at 200 req/s sustained showed p99 latency overhead under 4 ms versus in-memory.

## Consequences

- **Positive:** Rate limits behave correctly across replicas and survive deploys. No new infrastructure dependency.
- **Negative:** Postgres becomes a hot path for the incidents-API. Connection pool sizing matters under high QPS; the overlay pre-sets `pool_min=2, pool_max=10`.
- **Neutral:** Future stores (Redis, in-cluster KV) plug in behind the same Protocol without API changes.

## Related

- See [arch-08-deployment.md](../architecture/arch-08-deployment.md)
- Related ADRs: 028 (resilience), 030 (Helm chart)
