# Rate Limiting

> Rate limiting protects a system from being overwhelmed — by abusive clients, buggy retries, traffic spikes, or a
> noisy tenant — by capping how many requests a caller can make in a window. The interview signal is knowing the
> **four algorithms**, their trade-offs, and — the hard part — how to make a limiter **correct and fast across a
> distributed fleet** (where the counter is shared state).

**One-line mental model:** a rate limiter answers *"is this caller allowed to do one more thing right now?"* in
**O(1)** and sub-millisecond time, using a **shared, atomic counter** so every server agrees.

---

## Problem

Without limits, one client (or one bug, or one attacker) can consume all your capacity: exhaust DB connections,
blow your downstream API quota, run up cost, or starve other users. You need to **reject or delay** excess traffic
*before* it does damage — fairly, cheaply, and consistently across all your servers.

## When to use
- **Public APIs** — per-user/API-key quotas (e.g. 1000 req/hour).
- **Protect a scarce resource** — DB, a downstream 3rd-party API with its own quota, an expensive endpoint.
- **Abuse / DoS mitigation** — per-IP limits on login, signup, password reset.
- **Multi-tenant fairness** — stop one tenant's spike from starving others (noisy-neighbor).
- **Politeness** — a crawler limiting itself per host (see [web crawler](../02-systems/web-crawler/deep-dives.md#4-politeness-distributed-rate-limiting--robots)).

## When NOT to use / be careful
- **Internal, trusted, low-volume** calls where the limiter adds latency for no benefit.
- Don't use it as a substitute for **capacity planning or backpressure** — it's one layer, not the whole story
  (see [backpressure](queues-workers.md#backpressure-protecting-yourself-when-producers-outrun-consumers)).
- Over-aggressive limits hurt legitimate users → tune with real traffic + return good errors.

---

## Where it lives

- **API gateway / edge / load balancer** — the most common place; reject early before requests reach app servers.
  (See API gateway *(coming)*.)
- **A dedicated rate-limit service** — called by services that need shared limits.
- **In the app / middleware** — simplest, but each instance needs the *shared* counter to be correct fleet-wide.
- **Client-side** — polite self-throttling (crawler), not a security control (clients can lie).

The response for a rejected request: **HTTP `429 Too Many Requests`** + a **`Retry-After`** header and rate-limit
headers (`X-RateLimit-Limit/Remaining/Reset`) so well-behaved clients back off correctly.

---

## The four algorithms (know all four cold)

### 1. Fixed window counter
Count requests per fixed clock window (e.g. per minute). Reset the counter each window.

```text
key = user:123:minute:2026-09-25T14:03
count = INCR key ; EXPIRE key 60
allow if count <= limit
```
- ✅ Trivial, memory-cheap (one counter per window).
- ❌ **Boundary burst problem:** a client can send `limit` requests at 14:03:59 and another `limit` at 14:04:00 →
  **2× the limit** in a 2-second span. The window edge is exploitable.
- **Use when:** rough limits are fine and simplicity matters.

### 2. Sliding window log
Store a **timestamp per request**; count how many fall within the last `window` seconds (a rolling window).

```text
now = t; drop timestamps older than (t - window); add t; allow if count <= limit
```
- ✅ **Exact** — no boundary burst; precise rolling count.
- ❌ **Memory-heavy** — stores every request timestamp (bad at high volume). O(n) cleanup.
- **Use when:** precision matters and volume per key is modest.

### 3. Sliding window counter (the practical middle ground)
Approximate the sliding window using the current + previous fixed-window counts, weighted by overlap.

```text
count ≈ curr_window_count
      + prev_window_count × (overlap fraction of the previous window still in view)
```
- ✅ Smooths the boundary burst, **cheap** (two counters), good enough for almost everything.
- ❌ Slightly approximate (assumes even distribution within a window).
- **Use when:** you want sliding-window fairness without the log's memory cost. **Common production choice.**

### 4. Token bucket (the most versatile — and the one to default to)
A bucket holds up to `B` tokens; tokens **refill at rate `r`/sec**. Each request removes one token; if the bucket
is empty, reject (or wait).

```text
tokens = min(B, tokens + (now - last_refill) × r)
if tokens >= 1: tokens -= 1 ; allow
else: reject
```
- ✅ **Allows controlled bursts** (up to `B`) while enforcing a long-run average rate `r` → matches real traffic
  (people click in bursts). Memory = a couple of numbers per key. Smooth and intuitive.
- ❌ Two parameters to tune (`B` and `r`).
- **Use when:** almost always — it's the default for API rate limiting. Bucket size sets burst tolerance; refill
  rate sets sustained throughput.

### Leaky bucket (the sibling — smoothing, not just limiting)
Requests enter a **fixed-size queue** and are processed (leak) at a **constant rate**; overflow is dropped.
- ✅ **Smooths bursts into a steady output rate** — good when the *downstream* needs a constant, even flow.
- ❌ Adds latency (requests wait in the bucket); no burst allowance (that's the point).
- **Token bucket vs leaky bucket:** token bucket **allows bursts** up to `B` (limits the *average*); leaky bucket
  **enforces a smooth constant rate** (no bursts). Choose token bucket to cap usage while tolerating bursts; leaky
  bucket to protect a downstream that needs even pacing.

| Algorithm | Burst allowed? | Memory | Precision | Notes |
|---|---|---|---|---|
| Fixed window | yes (2× at edge) | tiny | rough | boundary-burst bug |
| Sliding log | no | **high** | exact | stores every timestamp |
| Sliding counter | limited | small | good approx | practical default for windows |
| **Token bucket** | **yes, ≤ B** | small | good | **default for APIs**; average + burst |
| Leaky bucket | no (smooths) | small | good | constant output rate; adds latency |

---

## Distributed rate limiting (the hard part — this is where the interview goes)

With many app servers, an **in-memory per-server counter is wrong**: if the limit is 100/min and you have 10
servers each allowing 100, the real limit is 1000/min. The counter must be **shared and atomic**.

### The standard solution: Redis
- Keep the counter/bucket in **Redis** (shared by all servers); every server does the check against it.
- **Atomicity is essential** — a naive `GET`-then-`SET` has a race (two servers read the same count and both
  allow). Use one of:
  - **`INCR` + `EXPIRE`** (fixed/sliding-counter) — `INCR` is atomic.
  - A **Lua script** that reads, computes, and writes the token-bucket state **in one atomic server-side op**
    (the common production approach — the whole check-and-decrement happens atomically in Redis).
  - Redis **sorted sets** for sliding-window log (`ZADD`/`ZREMRANGEBYSCORE`/`ZCARD`).

### The trade-off: accuracy vs latency/throughput
- **Central Redis = accurate** but every request pays a network round-trip and Redis is a hot dependency/SPOF.
- **Local + sync = fast** but approximate. Common hybrids:
  - **Local token bucket per server** with a share of the global budget (e.g. 10 servers → 10/min each for a
    100/min limit), periodically rebalanced. Fast, approximate, no per-request Redis hop.
  - **Local cache of the counter** synced to Redis every N ms — bounded inaccuracy for big throughput gains.
- **What if Redis is down?** Decide **fail-open** (allow traffic — availability over strict limiting; usual for
  quota limits) or **fail-closed** (reject — protection over availability; for abuse/DoS defense). State the choice.

> **Interview line:** *"I'd keep the token-bucket state in Redis and do the check-and-decrement in a single atomic
> Lua script so concurrent servers can't both consume the last token. If per-request Redis latency is too costly at
> scale, I'd shard the budget to per-server local buckets synced periodically — trading a little accuracy for
> throughput — and I'd decide fail-open vs fail-closed based on whether the limit is for quota or for abuse."*

---

## Scoping limits (name the dimensions)

Limits apply at different granularities, often **layered**:
- **Per-user / per-API-key** — the usual quota (`1000/hour`).
- **Per-IP** — abuse/DoS on unauthenticated endpoints (login, signup) — but beware shared NATs/proxies.
- **Per-tenant** — multi-tenant fairness + noisy-neighbor isolation; often tiered by plan.
- **Global** — protect a downstream/back-end absolute capacity, regardless of who.
- **Per-endpoint** — expensive endpoints get tighter limits than cheap ones.
- **Per-downstream-service** — cap your own calls to a 3rd-party within *their* quota.

You typically enforce **several at once** (per-user AND global AND per-endpoint) — the most restrictive wins.

---

## Architecture (typical)

```text
Client → API Gateway / edge ──check──► Rate-limit store (Redis: token bucket via Lua)
             │  allow → forward to service
             │  deny  → 429 + Retry-After + X-RateLimit-* headers
             ▼
         App services
```

- Check at the **edge/gateway** so rejected traffic never reaches app servers.
- Shared **Redis** holds the counters/buckets; atomic ops (Lua/INCR) guarantee fleet-wide correctness.
- Return proper `429` + headers so clients self-throttle.

---

## Trade-offs (argue both sides)

- **Accuracy vs performance:** central atomic counter (exact, +latency, hot dependency) vs local/approximate
  (fast, slightly wrong). Pick by how strict the limit must be.
- **Token vs leaky bucket:** allow bursts (token) vs enforce smooth rate (leaky). Match to whether the downstream
  tolerates bursts.
- **Fixed vs sliding window:** simplicity vs boundary-burst correctness; sliding-window *counter* is the pragmatic middle.
- **Fail-open vs fail-closed** on limiter outage: availability vs protection.
- **Where to enforce:** edge (reject early, but needs shared state at the edge) vs service (closer to the resource).

---

## Interview wording

> "I'd default to a token bucket — it enforces an average rate while allowing natural bursts up to the bucket size."
> "For a distributed fleet the counter must be shared and atomic — I'd use Redis with a Lua script so two servers
> can't both consume the last token."
> "Fixed windows have a boundary-burst bug where you get 2× the limit across the edge; I'd use a sliding-window
> counter or token bucket to avoid it."
> "I'd layer limits: per-user, per-IP for unauthenticated endpoints, and a global cap to protect the backend."
> "If Redis is down, for a usage quota I'd fail open to preserve availability; for abuse protection I'd fail closed."
> "On rejection I return 429 with Retry-After so well-behaved clients back off instead of hot-looping."

---

## Where it appears (across systems)

- **API gateway** — the canonical home of rate limiting (per-key quotas). See API gateway *(coming)*.
- **Web crawler** — politeness = per-host rate limiting, made local by host-sharding the frontier
  ([crawler politeness](../02-systems/web-crawler/deep-dives.md#4-politeness-distributed-rate-limiting--robots)).
- **URL shortener** — limit `POST /urls` per IP/account to stop bulk abuse.
- **Payment / auth systems** — per-IP limits on login/reset to block credential stuffing.
- **Multi-tenant SaaS** — per-tenant quotas + noisy-neighbor isolation.
- **Distributed Rate Limiter** is itself a full system (#7) — this pattern is its core.

---

## Quick checklist

- [ ] **Algorithm:** token bucket (default) / sliding-window counter / leaky (smoothing) — chosen with reason?
- [ ] **Distributed correctness:** shared **atomic** counter (Redis INCR/Lua), not per-server memory?
- [ ] **Scope:** per-user / per-IP / per-tenant / global / per-endpoint — layered, most-restrictive wins?
- [ ] **Limiter outage:** fail-open (quota) or fail-closed (abuse)?
- [ ] **Response:** `429` + `Retry-After` + `X-RateLimit-*` headers?
- [ ] **Accuracy vs latency** trade acceptable (central vs local buckets)?
- [ ] Tuned against real traffic so legit users aren't throttled?

← Back to **[patterns](../index.md)** · Related: [queues & workers (backpressure)](queues-workers.md) · [caching](caching.md)
