# Web Crawler — API & Data Model

> A crawler is mostly an **internal pipeline**, not a public API. The "interfaces" are between components (the
> frontier, fetchers, parsers, stores) plus a thin control/admin API. Model those, and the data stores each
> component owns.

---

## Control / admin API (thin)

```text
POST /v1/crawls              { "seeds": ["https://…"], "scope": {...}, "maxDepth": 10 }
     → 202 { "crawlId": "…" }                 # start/configure a crawl

GET  /v1/crawls/{id}/stats   → { pagesFetched, frontierSize, pagesPerSec, errors, dedupRate }

POST /v1/crawls/{id}/pause   /resume /stop     # operational control

POST /v1/urls                { "url": "…", "priority": 5 }   # inject a URL (recrawl / manual)
```

Everything else is component-to-component, and mostly **queue-based** (async) rather than request/response.

---

## Internal component interfaces

```text
Frontier.next()            -> URL to fetch (respecting priority + politeness)
Frontier.add(url, meta)    -> enqueue if not already seen / allowed by scope + robots
Fetcher.fetch(url)         -> (status, headers, body) | error
Parser.parse(page)         -> { links[], text, contentHash }
SeenSet.seen(url) / add(url)-> membership check + record (Bloom + backing store)
ContentStore.put(url, body)-> store raw page (object storage)
Robots.allowed(url)        -> bool (cached per host)
DNS.resolve(host)          -> IP (cached)
```

These are the seams you scale and fail-isolate independently.

---

## Data model — who stores what

The crawler owns several distinct datastores, each chosen for its access pattern.

### 1. URL Frontier (the queue of URLs to fetch)
Not one queue — a structured set giving **priority** and **politeness** (detailed in
[Deep Dives](deep-dives.md#1-the-url-frontier-the-heart-of-the-crawler)).

```text
frontier entry:
  url            (normalized)
  host           (for politeness routing)
  priority       (0..n; higher = crawl sooner)
  depth          (from seed; for depth limits / trap control)
  enqueued_at
  next_eligible_at   (for crawl-delay / recrawl scheduling)
```
- **Store:** disk-backed, **sharded by host** (so one host's URLs live together for politeness). A [queue +
  workers](../../01-patterns/queues-workers.md) with per-host sub-queues, persisted (Kafka/durable queue/DB) so a
  crash doesn't lose the backlog.

### 2. "Seen" / URL-dedup set
```text
seen(url_hash) -> exists?
```
- **Store:** **Bloom filter** in memory for the fast check (~1.2 GB per 1B URLs at ~1% FP), backed by a
  **sharded KV** (e.g. `url_hash → crawled_at`) for exactness and metadata. Sharded by `hash(url)`.
- This is [idempotency](../../01-patterns/idempotency.md) at web scale — "have I processed this key before?"

### 3. Content store (the fetched pages)
```text
page:
  url            (or url_hash as key)
  fetched_at
  status_code
  content_hash   (for content dedup)
  body_location  (pointer into object storage)
  headers, etag, last_modified   (for conditional recrawl)
```
- **Store:** **object storage (S3)** for the raw compressed HTML (big blobs, write-once) +
  a **metadata table** (KV/columnar) for the fields above, keyed by `url_hash`. See
  object storage & CDN *(coming)*.
- **Why object storage, not a DB:** pages are large immutable blobs read sequentially by downstream jobs — the
  classic object-storage access pattern. Don't put 100 KB blobs in your OLTP database.

### 4. Content-dedup index (near-duplicate detection)
```text
content_hash / simhash -> canonical_url
```
- **Store:** KV of exact hashes for exact dup; a **SimHash/MinHash** index for near-duplicates. Sharded.

### 5. `robots.txt` + DNS caches
```text
robots(host) -> parsed rules + fetched_at (TTL)
dns(host)    -> IP + TTL
```
- **Store:** [cache](../../01-patterns/caching.md) (Redis/local), TTL'd. Almost every fetch hits these → caching them
  is essential (both are per-*host*, and we crawl many pages per host).

### 6. Link graph (nice-to-have, if building ranking)
```text
edges: (from_url_hash, to_url_hash)
```
- **Store:** append-only edge list in object storage / a graph or columnar store, processed offline (e.g. for
  PageRank). Large; keep it off the hot crawl path.

---

## SQL or NoSQL? (per store — there's no single answer)

The crawler deliberately uses **different stores for different access patterns** — a good thing to say out loud:

- **Raw pages → object storage (S3).** Big immutable blobs. Not a database question.
- **Seen-set / URL metadata / content-hash index → NoSQL KV**, sharded by hash. Access is point lookup by key at
  massive scale with heavy writes — the KV sweet spot. A relational DB would buckle on the write volume + size.
- **Frontier → a durable queue** (Kafka / a purpose-built queue / disk-backed structure), not a general DB.
- **Crawl config / admin → a small SQL DB** is fine (low volume, relational).

> **Interview line:** *"There's no single database here. Raw pages go to object storage; the seen-set and URL
> metadata to a sharded KV store because it's write-heavy point-lookup at billions of keys; the frontier is a
> durable, host-sharded queue; and the little bit of admin/config state can live in SQL. Choosing per access
> pattern is the point."*

---

## URL normalization (don't skip — it's a dedup correctness issue)

Before dedup/enqueue, **normalize** URLs so trivially-different strings map to one entry:
- lowercase scheme + host; remove default ports; resolve `.`/`..`; sort or strip tracking query params
  (`utm_*`); decode/encode consistently; strip fragments (`#…`); handle `http` vs `https` and `www` vs apex per policy.

→ **Why it matters:** without normalization, `example.com/a`, `example.com/a/`, `EXAMPLE.com/a?utm=x#top` look
like four URLs → you crawl the same page repeatedly and pollute the seen-set. Normalization is the front door to
good dedup.

→ Next: **[Architecture](architecture.md)**
