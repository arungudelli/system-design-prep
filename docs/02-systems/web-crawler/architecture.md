# Web Crawler — Architecture

## 9. Simplest architecture (then we scale it)

Start with the crawl loop as a single pipeline, then distribute it.

```text
     Seeds
       |
   [ URL Frontier ]  ◄─────────────── new links (deduped)
       |  next()                              ▲
       ▼                                      │
   [ Fetcher ] ──http──► the web              │ extract + normalize + dedup
       |  (robots + DNS + politeness)         │
       ▼                                      │
   [ Parser / Extractor ] ───────────────────┘
       |
       ▼
   [ Content Store ]  (object storage + metadata)
```

**The crawl loop:** frontier hands out a URL → fetcher downloads it (after checking `robots.txt`, resolving DNS,
respecting politeness) → parser extracts links + content → new links are normalized and dedup-checked → unseen
ones go back into the frontier → content is stored. Repeat forever.

**Every box justified:**
- **URL Frontier** — the prioritized, polite queue of what to crawl next. The heart of the system.
- **Fetcher** — I/O-bound downloader; many concurrent connections; enforces politeness + robots + DNS.
- **Parser/Extractor** — pulls links + text, computes content hash, normalizes URLs.
- **Content Store** — durable home for pages (object storage) + metadata.

---

## The full distributed architecture (what you present)

```text
                         ┌──────────────────────────────────────────┐
        Seeds ─────────► │            URL FRONTIER                   │
                         │  front queues (priority) → back queues    │◄──── new deduped URLs
                         │  (per-host, politeness) ; sharded by host │
                         └──────────────┬───────────────────────────┘
                                        │ next() (host-affinity)
              ┌─────────────────────────┼─────────────────────────┐
              ▼                         ▼                          ▼
        [ Fetcher pool ]          [ Fetcher pool ]           [ Fetcher pool ]   (async I/O, many hosts each)
              │  uses:  DNS cache  •  robots.txt cache  •  per-host rate limiter
              ▼
        [ Parser / Extractor pool ]
              │  extract links → normalize → SEEN-SET check (Bloom + KV)
              │  compute content hash → CONTENT-DEDUP check (SimHash)
              ├──────────────► Content Store: raw HTML → Object Storage (S3, compressed)
              │                              metadata → sharded KV
              └──────────────► unseen, in-scope links → back to FRONTIER
                               (+ link edges → graph store, optional)
```

- **Frontier** is sharded (usually **by host**) so a host's URLs and its politeness state live together.
- **Fetchers** pull from the frontier with **host affinity**, run thousands of async connections, and consult the
  **DNS cache**, **robots cache**, and a **per-host rate limiter** before each request.
- **Parsers** dedup URLs (Bloom+KV) and content (hash/SimHash), store pages to object storage, and feed new URLs back.
- Components are **stateless workers** around **shared, sharded state** (frontier, seen-set, stores) → scale each independently.

---

## 10. Request / data flows

### Main crawl flow
```text
1. Frontier.next() → URL (highest priority host that is politeness-eligible now)
2. Fetcher: robots.allowed(url)? (cached)   → if disallowed, drop
3. Fetcher: DNS.resolve(host) (cached) → connect → GET (conditional if we have etag/last-modified)
4. On 200: pass body to Parser; on 3xx: enqueue redirect target; on 4xx/5xx: record + backoff policy
5. Parser: extract links, normalize each
6. For each link: SeenSet.seen()? → if new AND in-scope AND robots-allowed → Frontier.add()
7. Parser: contentHash → ContentStore only if not a known duplicate
8. Update metadata (fetched_at, status, etag); schedule recrawl if applicable
```

### Politeness path (per host)
```text
Before fetch: check per-host limiter (≤1 in-flight / respect crawl-delay).
If not eligible yet → the host's back-queue holds its URLs until next_eligible_at.
```

### Failure path (examples)
```text
Fetch timeout/5xx → retry with backoff up to N, then park/drop the URL
Fetcher crash     → in-flight URLs (leased from frontier) become visible again → re-fetched (idempotent: dedup handles it)
```

---

## 11. Bottlenecks (ranked)

| Rank | Bottleneck | Why | Detect via | Fix |
|---|---|---|---|---|
| 1 | **Network bandwidth / fetch concurrency** | It's I/O-bound at 100 KB × 400/s+ | throughput, saturated NICs, fetch latency | more fetchers, async I/O, distribute across regions/links |
| 2 | **DNS resolution** | one lookup per host, on the hot path | DNS latency, resolver QPS | **DNS cache** + async resolver + local resolvers |
| 3 | **Frontier throughput / contention** | every crawl step reads+writes it | frontier op latency, queue depth | **shard by host**, disk-backed, batch ops |
| 4 | **Seen-set lookups** | billions of checks | dedup latency | **Bloom filter** in memory + sharded KV backing |
| 5 | **Storage write throughput** | 40 MB/s+ of pages | write latency, S3 throttling | object storage (scales), compress, batch/multipart |
| 6 | **Politeness ceiling (per host)** | can't exceed a host's tolerance | per-host 429/blocks | crawl **breadth** across many hosts, not depth on few |

Note the profile: it's about **throughput and coordination**, not read latency. Politeness is a *hard ceiling* you
design *around* (breadth), not something you "scale away."

---

## 12. Scaling each bottleneck

- **Throughput** → add **fetcher machines**, each running thousands of **async** connections; distribute across
  network egress points/regions. Fetchers are stateless → scale horizontally.
- **DNS** → aggressive **[caching](../../01-patterns/caching.md)** (per host, TTL'd) + async resolution + run your own
  caching resolvers to avoid hammering upstream DNS.
- **Frontier** → **shard by host** across nodes; disk-backed queues; keep politeness state local to each shard.
- **Seen-set** → **Bloom filter** (memory-efficient) fronting a **sharded KV**; shard by `hash(url)`.
- **Storage** → **object storage** (effectively infinite), compress on write, batch/multipart uploads; tier old data.
- **Politeness** → maximize *global* throughput by spreading load over **many hosts** simultaneously; you cannot
  exceed one host's rate, so coverage comes from breadth + prioritization.

---

## 20. Evolution with scale

| Stage | Shape | What forced the jump |
|---|---|---|
| **1 · Small** | single-process loop: in-memory frontier + set, fetch, store to disk | — |
| **2 · Moderate** | fetcher/parser worker pools + durable frontier + Bloom seen-set + object storage | one process can't hit throughput; crash lost the frontier |
| **3 · Large** | frontier **sharded by host** + fetchers with host affinity + DNS/robots caches + content dedup | frontier contention; DNS bottleneck; duplicate content |
| **4 · Extreme** | multi-region fetchers, prioritized + freshness-aware recrawl scheduler, JS-render tier for select sites | bandwidth/geography limits; freshness needs; SPA sites |

> The transitions are driven by **throughput, coordination, and dedup at scale** — very different triggers from a
> read-heavy service. Multi-region here is about **bandwidth/egress and crawling geographically-close sites**, not
> user latency.

→ Next: **[Deep Dives](deep-dives.md)**
