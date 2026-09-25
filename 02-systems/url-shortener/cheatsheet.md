# URL Shortener — Cheatsheet

> One-page revision. If you can reproduce this from memory, you can pass the interview.

---

## Problem
Long URL → short code; visiting the code redirects to the original. Fast, at scale, forever.

## Requirements
- **Must:** create code, redirect, uniqueness.
- **Defer:** custom alias, expiry, analytics, accounts, malicious-URL detection.
- **Dominant NFRs:** redirect **p99 < 50 ms**, read availability **99.99%**, never lose a mapping.
- Reads ≫ writes (**~100:1**); codes are **immutable**; eventual consistency OK for reads.

## Numbers (memorize)
| | |
|---|---|
| New URLs | 100M/mo → **40 w/s avg, 120 peak** |
| Reads | 100:1 → **4k/s avg, 12k peak** |
| Storage | 500 B × 100M/mo × 12 ≈ **600 GB/yr** (fits 1 node for years) |
| Cache | hot set ~**10 GB** → **90%+ hit** |
| Key length | **7 base62** = 62⁷ ≈ **3.5T** codes |
| Bandwidth | negligible |

**Conclusion the numbers force:** reads dominate → cache + replicas; writes/storage trivial → **no sharding**.

## API
```text
POST /v1/urls {longUrl, customAlias?, ttlDays?}  -> 201 {shortCode, shortUrl}
GET  /{shortCode}                                -> 302 Location: <longUrl>
Idempotency-Key on POST to make retries safe.
```

## Data model
```text
urls: short_code (PK) | long_url | created_at | expires_at? | owner_id?
```
Access pattern = **keyed point lookup, no joins** → KV store *or* SQL both fit (pick by team + need for
constraints). PK = short_code.

## Architecture
```text
Client → LB → [stateless API servers] → cache (Redis, cache-aside)
                                       → DB (primary + read replicas)
create path also → Key Generation Service (counter+base62, leased ranges)
redirect → 302; click event → queue (async analytics)
```

## Key generation (the core sub-problem)
- **Counter + base62**, handed out in **disjoint ranges** by a **KGS** → collision-free, no hot-path
  coordination, no DB read on create.
- Crashed server's range → **skip it** (never reuse codes).
- Unguessable? Run counter through a **reversible permutation**.
- Avoid hash-truncation (collision → read+retry every create).

## Caching
- **Cache-aside** reads + **write-through** on create; long TTL (immutable codes) → 90%+ hit; **LRU**.
- **Redis down** → fall through to DB (degrade, not fail).
- **Stampede** → single-flight + jittered TTL + pre-warm. **Penetration** → cache negatives / Bloom filter.
- **Hot key** (viral link) → local in-proc cache + replicate key + CDN.

## 301 vs 302
- **302 (default):** every click reaches you → **analytics + control**; more traffic.
- **301:** browser/CDN caches → **less load/latency**; **lose analytics + control**.

## Bottlenecks (ranked)
1. Read throughput → cache → replicas → CDN/edge
2. Hot key → local cache / replicate / CDN
3. Code generation → KGS ranges
4. Writes/storage → **not a problem** (say it)

## Failure one-liners
- Cache down → DB fallback (+ stampede guard). DB primary down → **reads OK (cache/replicas)**, creates pause.
- Viral link → hot-key mitigations. Downstream slow → timeout+breaker+bulkhead; **keep redirect path external-call-free**.
- Bad deploy → canary + auto-rollback (redirect handler = crown jewel). AZ fail → multi-AZ. Region fail →
  multi-region (easy: immutable mapping; partition KGS per region).

## Evolution
1 server+DB → +cache+replicas → +KGS ranges+CDN → +multi-region reads.

## The 302-word summary line
> "Read-heavy 100:1, sub-50ms redirects → cache-first (Redis cache-aside, long TTL, immutable codes) + read
> replicas + single primary for trivial writes. Codes = counter→base62 via a KGS leasing ranges (collision- &
> contention-free). Default 302 for analytics/control. SQL or KV both fit the keyed lookup. No sharding —
> storage/bandwidth are non-issues; reads are the only scaling axis, handled by cache/replicas/CDN."

← Back to **[README](02-systems/url-shortener/README.md)**
