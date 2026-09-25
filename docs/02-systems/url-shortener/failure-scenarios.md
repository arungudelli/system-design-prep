# URL Shortener — Failure Scenarios

> Seniors design the happy path. For each failure: **how do we detect it, contain the blast radius, and
> recover?** The theme here: **reads must survive almost anything** (broken redirects = every link is dead);
> creates can tolerate brief downtime.

---

## Cache (Redis) fails

- **Impact:** all reads fall through to DB / read replicas. Latency rises; correctness intact.
- **Detect:** cache hit ratio drops to 0, Redis health check fails, DB read QPS spikes.
- **Contain:** DB + read replicas must be sized to absorb at least a burst of full read load. Circuit-break to
  DB cleanly (don't block on a dead cache — short connect timeout).
- **Recover:** cache comes back cold → guard against **stampede**: request coalescing (single-flight per key),
  jittered TTLs, and pre-warm the top-N hot codes on restart.
- **Related:** **cache penetration** — floods of unknown codes bypass cache; cache negative results + optional
  Bloom filter so a dead cache doesn't turn into a DB-killing miss storm.

## DB primary fails

- **Impact:** **writes (creates) pause** until failover; **reads continue** from cache + read replicas.
- **Detect:** write errors, replication heartbeat lost, failover alarm.
- **Contain:** creates return `503` + `Retry-After` (acceptable — create availability target is only 99.9%).
  Reads are unaffected because they're served from cache/replicas.
- **Recover:** automated failover promotes a replica (RTO seconds). Ensure the KGS's range state is durable so
  code generation resumes without reissuing ranges.

## A read replica fails

- **Impact:** minor — LB/routing drops it; remaining replicas + cache absorb load.
- **Detect:** replica health check, replication lag climbing on survivors.
- **Contain/recover:** autoscale/replace the replica; cache shields the primary in the meantime.

## API server dies

- **Impact:** none visible — it's stateless; LB health-checks it out and routes around.
- **Contain:** run N+ servers across AZs; autoscale on CPU/QPS.
- **KGS note:** a dead server "loses" its unused counter range → we simply **skip those codes** (never reuse).

## Key Generation Service fails

- **Impact:** servers keep minting from their **already-leased local range**, so creation continues for a while
  even if KGS is down. Only when a server exhausts its range and can't lease a new one do creates start failing.
- **Detect:** KGS health, "range exhaustion imminent" metric per server.
- **Contain:** lease **generous ranges** and refill **early** (at, say, 20% remaining) so a brief KGS outage is
  invisible. Make KGS HA (its state is tiny: "next range").
- **Recover:** KGS restart reads persisted "next range" and resumes — never re-hand a range already given out.

## Hot key (a link goes viral)

- **Impact:** one cache node / one code gets a huge share of reads → that node saturates.
- **Detect:** per-key hit rate, single Redis node CPU/network hot, uneven latency.
- **Contain:** **local in-process cache** on each API server for top-N codes (coalesces requests), **replicate
  the hot key** across cache nodes, and/or let a **CDN** absorb it at the edge.
- This is a read-amplification problem, not a data problem — the value is tiny; it's the *request rate* that hurts.

## Downstream slow (e.g. safe-browsing check on create)

- **Impact:** if the create path calls an external checker synchronously and it slows (200 ms → 10 s), create
  latency and thread pools suffer.
- **Contain:** **timeout + circuit breaker + bulkhead** around the external call; on breaker-open, either
  fail-closed (reject create) or fail-open (accept + async re-scan) per risk appetite. Keep the **redirect path
  free of external calls** entirely.

## Queue (analytics) unavailable

- **Impact:** none to the user — analytics is off the critical path. We may lose/delay click events.
- **Contain:** buffer locally with bounded size + drop-oldest (analytics tolerates loss); backpressure never
  reaches the redirect. **Never let analytics degrade redirects.**

## AZ fails

- **Impact:** lose capacity in that AZ.
- **Contain:** multi-AZ deployment for API servers, cache, and DB (primary + replica in different AZs) → survive
  with reduced capacity; autoscale in healthy AZs.

## Region fails

- **Impact:** total outage if single-region.
- **Contain:** multi-region for the read path is **cheap here** (immutable mapping → replicate everywhere).
  DNS/anycast failover for reads; key generation uses **per-region counter partitions** so failover can't
  produce colliding codes. Only invest in this if regional resilience is a requirement.

## Bad deploy

- **Impact:** a broken redirect handler breaks *every* link → highest-severity failure mode.
- **Contain:** **canary** a small % of traffic first; watch p99 + error rate + redirect success; **auto-rollback**
  on regression. Keep the redirect handler tiny and boring — it's the crown jewel; minimize churn there.
- **Schema changes:** must be backward-compatible (expand/contract) so old and new app versions coexist during
  the rollout.

---

## Failure-handling toolkit used here

`timeout` · `retry + backoff + jitter` · `idempotency key (create)` · `circuit breaker` (safe-browsing) ·
`bulkhead` · `graceful degradation` (cache down → DB) · `stampede protection` (single-flight, jittered TTL) ·
`negative caching / Bloom filter` (penetration) · `canary + auto-rollback` · `multi-AZ`.

## Priorities (say this)

> *"My availability budget goes to the **read/redirect path** — it must survive cache loss, replica loss, AZ
> loss, and bad deploys, because a broken redirect breaks every existing link. Creates can degrade to a retry
> during a primary failover; that's an acceptable, much rarer, and lower-severity failure."*

→ Next: **[Interview Questions](interview-questions.md)**
