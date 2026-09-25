# URL Shortener

> Design a service like TinyURL / Bit.ly: take a long URL, return a short code, and redirect anyone who
> visits the short code back to the original — fast, at scale, essentially forever.

This is the canonical "start simple" system. It looks trivial, and the naive design *is* trivial — which is
exactly why it's a great interview: the signal is in how you handle **read scale, latency, unique key
generation, and the 301-vs-302 redirect trade-off**, not in drawing a hundred boxes.

---

## Why it's a good first system

- **Read-heavy** (≈100:1) → forces the cache + read-replica conversation, the most common scaling story.
- **Tiny writes** → teaches you to *not* shard prematurely (the math says one DB is plenty for years).
- **Unique key generation** → a genuinely interesting distributed-systems sub-problem (counter vs hash vs KGS).
- **Latency-sensitive redirect** → cache-first design, and a subtle caching-vs-analytics trade-off (301 vs 302).

---

## Read in this order

1. **[Requirements](requirements.md)** — problem statement, clarifying questions, functional + NFRs
2. **[Capacity](capacity.md)** — the math that decides the architecture
3. **[API & Data Model](api-data-model.md)** — endpoints, schema, key-length math
4. **[Architecture](architecture.md)** — simplest design → request flows → scaling
5. **[Deep Dives](deep-dives.md)** — key generation, caching, 301 vs 302, custom aliases, analytics, expiry
6. **[Trade-offs](tradeoffs.md)** — the decisions and both sides of each
7. **[Failure Scenarios](failure-scenarios.md)** — what breaks and how we contain it
8. **[Interview Questions](interview-questions.md)** — attempt first, then reveal the model answer
9. **[Cheatsheet](cheatsheet.md)** — 1-page revision
10. **[★ Principal Deep Dive](principal-deep-dive.md)** — the exhaustive **high-scale** variant (10B reads/mo, 99.999%, <20 ms): ZooKeeper KGS, Kafka→Flink→ClickHouse analytics, consistent-hashing sharding, anycast/CDN, full 5/10-yr capacity math

---

## The 30-second version (know this cold)

- **Scale:** ~100M new URLs/month → ~40 writes/s, ~4k reads/s (peak ~120 w/s, ~12k r/s). Reads ≫ writes.
- **Storage:** ~500 B/record → ~600 GB/year. One DB node holds years of data.
- **Key:** 7 chars of base62 = 62⁷ ≈ **3.5 trillion** codes — plenty. Generate via a **counter + base62**
  (or a Key Generation Service), not by hashing-and-hoping.
- **Store:** KV lookup by `shortCode` → a key-value store (DynamoDB/Redis) or a simple indexed SQL table both work.
- **Read path:** `GET /{code}` → **cache** (Redis, ~90%+ hit) → DB → **302 redirect**.
- **Redirect code:** **302** (temporary) to keep control + analytics; **301** (permanent) only if you want
  browsers/CDNs to cache and you don't need per-click tracking.
- **Bottleneck:** read throughput → cache + read replicas. Writes and storage are never the problem here.
