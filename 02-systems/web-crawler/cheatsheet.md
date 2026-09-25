# Web Crawler — Cheatsheet

> One-page revision. Reproduce this from memory and you can drive the interview.

---

## Problem
Seed URLs → fetch pages → extract links → follow them → store content. Politely, at scale, refreshed over time.

## Profile (say first)
**Throughput/bandwidth-bound + coordination-heavy** — NOT read-latency. Opposite of the URL shortener.

## Requirements
- **Must:** fetch, extract+follow, **URL dedup**, **politeness (robots + per-host rate)**, store content.
- **Defer:** content dedup detail, freshness, JS rendering, priority, link graph, media.
- **Dominant NFRs:** max **global throughput** subject to **per-host politeness**; robust + resumable.

## Numbers (1B pages/month example)
| | |
|---|---|
| Fetch rate | 1B/mo ÷ 2.5M s ≈ **400/s** (peak ~1.2k) |
| Bandwidth | 400 × 100 KB ≈ **~320 Mbps–1 Gbps** (first-class cost) |
| Storage | ~100 TB/mo raw → **~20–30 TB/mo compressed** → object storage |
| Seen-set | billions of URLs → **Bloom filter ~1.2 GB/1B** |
| DNS | 400 lookups/s naive → **cache it** |

## Core components
```text
Frontier → Fetcher → Parser/Extractor → Content Store
   ▲            (robots, DNS, per-host rate)     │
   └──────── new deduped URLs ───────────────────┘
```

## URL Frontier (the heart) — priority + politeness
- **Front queues** = priority (crawl important first).
- **Back queues** = politeness, **one host per queue** → ≤1 connection/host.
- **Heap of per-host next-eligible times** → honor crawl-delay.
- **Shard by host** → politeness is a **local** decision (no distributed lock). **Durable** (never lose it).

## Dedup
- **URLs:** **Bloom filter** (no false negatives; rare FP = skip a new URL) + sharded KV backing. **Normalize first.**
- **Content:** exact = content hash; **near-dup = SimHash/MinHash**.

## Politeness (3 parts)
1. **robots.txt** — per host, **cached** w/ TTL; obey Disallow + Crawl-delay.
2. **Per-host rate** — ≤1 in-flight; enforced by one-host-per-back-queue.
3. **Identify** — descriptive User-Agent + contact.

## DNS (hidden bottleneck)
Cache per host + async resolve + own caching resolvers + serve stale on outage.

## Traps & robustness
Per-host **URL budget + depth limit** · param-explosion detection · **session-id stripping** · **max page size** ·
**request timeout** · **redirect cap** · tolerant parsing (never crash a worker on bad HTML).

## Freshness
**Adaptive recrawl** by change rate + **conditional GET** (`If-Modified-Since`/etag → **304** ≈ free) + sitemap `lastmod`.

## Storage
Raw pages → **object storage (S3), compressed**; metadata (url_hash, status, etag, content_hash, body ptr) → **KV**;
links → graph store (offline). **No single DB** — choose per access pattern.

## Bottlenecks (ranked)
1. Bandwidth/fetch concurrency → more async fetchers
2. DNS → cache
3. Frontier → shard by host
4. Seen-set → Bloom + KV
5. Storage → object store + compress
6. **Politeness = hard ceiling** → gain throughput via **breadth (many hosts)**, not depth

## Failure one-liners
- Fetcher dies → **leased** URLs redelivered; double-fetch harmless (dedup).
- **Frontier dies → scariest** (= lost discovery) → durable + replicated per shard; restore from checkpoint.
- Seen-set lost → snapshot Bloom + KV backing; cold = temporary re-fetch spike.
- Slow/hostile server → timeout + size cap + redirect cap; one host can't stall others.
- 429/403 → per-host backoff + Retry-After + park.
- Storage/indexer slow → **backpressure** to fetchers; decouple via durable queue.

## Patterns used (name them)
[queue+workers](01-patterns/queues-workers.md) (frontier) · [idempotency](01-patterns/idempotency.md) (dedup) ·
[sharding](01-patterns/sharding.md) (by host / url-hash) · [caching](01-patterns/caching.md) (DNS/robots) ·
object storage (pages) · rate limiting (politeness) · backpressure.

## JS rendering
Plain HTML default; **targeted headless render tier** only where needed (**10–100× cost**).

## Summary line
> "Throughput/coordination-bound. Two-level frontier (priority front + one-host-per back queues) sharded by host so
> politeness is local. Stateless async fetchers with cached DNS+robots. Bloom-filter URL dedup + SimHash content
> dedup. Pages → compressed object storage, metadata → KV. Trap budgets + timeouts for robustness; adaptive
> conditional-GET recrawls for freshness. Scale by adding fetchers/shards; politeness is the ceiling I design
> breadth around."

← Back to **[README](02-systems/web-crawler/README.md)**
