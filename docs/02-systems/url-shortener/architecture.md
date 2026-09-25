# URL Shortener — Architecture

## 9. Simplest architecture that meets the requirements

Start boring. At 120 writes/s and 12k reads/s, you do **not** need much.

```text
        Client (browser / API caller)
              |
        Load Balancer
              |
   [ Stateless API servers ]   (create + redirect handlers)
              |
          Database            (short_code -> long_url)
```

**Every box justified:**
- **Load balancer** — spread traffic across API servers, health-check them out on failure.
- **Stateless API servers** — hold no per-request state, so we can add/remove them freely and autoscale.
  They handle both `POST /urls` (create) and `GET /{code}` (redirect).
- **Database** — durable store of `short_code → long_url`. Single primary is fine for this write volume.

This design already satisfies the functional requirements. Now we make the read path fast and resilient.

---

## 10. Request / data flows

### Write flow (create) — synchronous, ~120/s

```text
1. Client → POST /v1/urls { longUrl }
2. API server validates + (optionally) safe-browsing check
3. API server obtains a unique short_code
      (counter+base62 via a Key Generation Service — see Deep Dives)
4. INSERT (short_code, long_url, created_at, expires_at) into DB
5. (write-through) put short_code → long_url into cache
6. Return 201 { shortCode, shortUrl }
```

### Read flow (redirect) — the hot path, must be <50 ms, ~12k/s

```text
1. Client → GET /{shortCode}
2. API server looks up cache (Redis)     ── HIT (~90%) → step 5
3.   on MISS → read DB by primary key
4.   populate cache (cache-aside)
5. Return 302 Found, Location: <longUrl>
6. (async, off critical path) emit a click event for analytics
```

Key point to say out loud: **the redirect is synchronous and latency-critical; the analytics write is not on
the critical path, so it's fire-and-forget/async.** Never make the user wait on analytics.

### Failure flow (example)

```text
Cache down  → reads fall through to DB (higher latency, still correct); guard against stampede on refill
DB primary down → reads still served from cache + replicas; creates pause until failover
```

---

## The design after adding the cache (the one you actually present)

```text
                         Client
                           |
                     Load Balancer
                           |
              [ Stateless API servers ]
                     |            \
                     |             \  (create) obtains code from
                cache lookup        \--->  Key Generation Service (counter+base62)
                     |
             +-------+-------+
             |               |
        Cache (Redis)    Database (primary + read replicas)
        90%+ hit         durable source of truth
```

- **Cache (Redis), cache-aside**: absorbs the 12k/s read load; `shortCode → longUrl`. Immutable codes → long
  TTLs → high hit ratio. On create, write-through so the new code is instantly hot.
- **Read replicas**: serve the ~10% cache misses; take read load off the primary.
- **Key Generation Service (KGS)**: hands out unique codes so API servers never collide. (Deep-dived next.)

---

## 11. Bottlenecks (ranked)

| Rank | Bottleneck | Why | Detect via | Fix |
|---|---|---|---|---|
| 1 | **Read throughput** | 12k/s of lookups | p99 read latency, DB CPU, cache hit ratio | **cache** → **read replicas** → CDN/edge |
| 2 | **Hot key** (viral link) | one code = huge share of reads | per-key hit rate, one Redis node hot | local in-process cache in front, replicate the key, CDN it |
| 3 | **Unique code generation** | collisions/contention if naive | duplicate-key errors, KGS latency | KGS with pre-allocated ranges (below) |
| 4 | **DB write path** | only ~120/s | write latency | basically never a problem at this scale |

Note what's *not* on the list: storage and bandwidth. The numbers ([Capacity](capacity.md))
proved they're non-issues. Saying that explicitly is good signal — you're scaling the real bottleneck, not a phantom.

---

## 12. Scaling each bottleneck

### Reads (the main story)
1. **Cache-aside with Redis** — first and biggest win; ~90% of reads never touch the DB.
2. **Read replicas** — serve cache misses; scale reads horizontally.
3. **CDN / edge** — for very high scale, resolve popular codes at the edge (works cleanly with `301`, or with
   short-TTL edge caching for `302`). Puts the redirect physically close to users → lower latency + less origin load.

### Hot keys
A single viral link can hammer one Redis node. Mitigate with a small **in-process (local) cache** on each API
server for the top-N codes (request coalescing), and/or **replicate the hot key** across cache nodes, and/or
let the **CDN** absorb it. See [caching pattern](../../01-patterns/caching.md) *(coming)*.

### Code generation
Use a **Key Generation Service** that hands each API server a **pre-allocated range** of counter values
(e.g. server A owns 1–1M, server B owns 1M–2M). Each server converts its next integer to base62 locally →
**zero coordination on the hot path, zero collisions.** Detailed in [Deep Dives](deep-dives.md).

---

## 20. Evolution with scale

| Stage | Shape | What forced the jump |
|---|---|---|
| **1 · Small** | 1 API server + 1 DB | — |
| **2 · Moderate** | LB + N stateless servers + **Redis cache** + read replicas | read QPS climbing; DB read latency |
| **3 · Large** | + **KGS with ranges** + **CDN/edge** for popular codes | code-gen contention; hot keys; global latency |
| **4 · Extreme** | + multi-region (active-active for reads, single-writer or partitioned key ranges) | single-region latency ceiling / regional resilience |

> Multi-region for a URL shortener is mostly about **read latency and resilience**, not write volume. Because
> codes are immutable, replicating the read-only mapping globally is *easy* — a rare case where going global is
> genuinely simple. You'd still coordinate **key generation** (partition the counter space per region so codes
> never collide).

→ Next: **[Deep Dives](deep-dives.md)**
