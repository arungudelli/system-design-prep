# Web Crawler — Capacity Estimation

> See the method in **[Capacity estimation](../../00-framework/capacity-estimation.md)**. For a crawler the numbers
> that matter are **pages/sec** (→ fetcher count + bandwidth), **storage** (→ object store + cost), and the
> **frontier/dedup size** (→ how you store "seen" URLs at billions-of-items scale).

---

## Assumptions (state them)

- Target: **1 billion pages / month** (a mid-scale crawl; real Google is ~trillions, but 1B shows the reasoning).
- Average page size: **~100 KB** of HTML (say so — it drives bandwidth + storage).
- Each page yields **~10 new links** on average, most already seen.
- Continuous crawl; peak ≈ 2–3× average.

---

## Fetch rate (pages/sec) → fetchers + bandwidth

```text
1B / month ÷ 2.5M seconds/month ≈ 400 pages/sec  (average)
peak ≈ 400 × 3                  ≈ 1,200 pages/sec
```

→ **So what?** This sets **how many concurrent fetchers** we need and the **bandwidth**. Fetching is
**I/O-bound** (waiting on remote servers), so one machine handles many concurrent connections.

**Fetcher/thread math:** if a fetch takes ~500 ms end-to-end (DNS + connect + download), one worker does ~2
pages/s; to hit 400/s you need ~200 concurrent fetch slots (more at peak). With async I/O, a handful of machines
running thousands of concurrent connections each covers this. **Politeness caps per-host concurrency**, so
throughput comes from **breadth across many hosts**, not hammering a few.

---

## Bandwidth

```text
400 pages/s × 100 KB ≈ 40 MB/s ≈ 320 Mbps (average)
peak ≈ ~1 Gbps
```

→ **So what?** **Ingress bandwidth is a first-class cost + bottleneck.** At larger scale (10×/100×) this becomes
multiple Gbps sustained → you distribute fetchers across regions/network links, and bandwidth/egress cost is a
top line item. (Contrast the URL shortener, where bandwidth was negligible.)

---

## Storage

```text
raw HTML:  1B/month × 100 KB = 100 TB/month  (before compression)
compressed (~3–5×): ~20–30 TB/month
per year:  ~250–350 TB/year compressed
```

→ **So what?** **Object storage (S3-style), not a database** — this is large, write-once, sequential-read blob
data. Compress on write (gzip/zstd). Storage cost grows linearly and unbounded with a continuous crawl → you
need **retention/tiering** (hot recent pages, cold archive) and this is a major cost driver.

---

## Frontier & "seen" set (the interesting one)

We must remember every URL we've **already seen** to avoid re-crawling. Over a year that's billions of URLs.

```text
Suppose 10B distinct URLs seen. Storing full URLs (~100 bytes) = ~1 TB just for the set.
A hash (say 8 bytes) per URL = ~80 GB. Still large for pure in-memory.
```

→ **So what?** A **Bloom filter** represents membership in ~**1.2 GB for 1B URLs at ~1% false-positive**
(~10 bits/element) — orders of magnitude smaller than storing URLs. Use a Bloom filter for the fast "seen?"
check (backed by a persistent store for exactness). This is *why* Bloom filters show up in every crawler design.
See [Deep Dives](deep-dives.md#2-url-deduplication-the-seen-set).

The **frontier itself** (URLs queued to fetch) is bounded by your in-flight backlog, not the whole web — but at
billions it's still disk-backed and sharded, not a single in-memory queue.

---

## DNS

Every fetch needs a hostname → IP resolution. At 400/s, naive per-fetch DNS would be **400 DNS lookups/s** and
add latency to every fetch.
→ **So what?** **DNS is a hidden bottleneck** → cache resolutions aggressively (many URLs share a host) and
resolve asynchronously. Caching turns 400 lookups/s into a trickle (one per host per TTL).

---

## Summary — what the numbers told us

| Number | Value | Design consequence |
|---|---|---|
| Fetch rate | ~400/s avg, ~1.2k peak | Async I/O fetchers; scale by adding fetchers/shards |
| Bandwidth | ~320 Mbps–1 Gbps | First-class cost + bottleneck; distribute fetchers |
| Storage | ~20–30 TB/month compressed | **Object storage** + compression + tiering |
| Seen-set | billions of URLs | **Bloom filter** (~1.2 GB/1B) + backing store |
| DNS | 400 lookups/s naive | **DNS cache** + async resolution |

The profile: **throughput/bandwidth-bound, storage-heavy, coordination-heavy** — nothing like the read-latency
game of the URL shortener. Design accordingly.

→ Next: **[API & Data Model](api-data-model.md)**
