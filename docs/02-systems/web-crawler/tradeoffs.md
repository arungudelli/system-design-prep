# Web Crawler — Trade-offs

> Dominant requirements: **maximize global throughput subject to per-host politeness**, at web scale, robustly.
> Every trade-off is judged against those.

---

## 1. BFS vs DFS vs priority crawl order

| | BFS (breadth-first) | DFS (depth-first) | Priority (best-first) |
|---|---|---|---|
| Behavior | crawl level by level | follow one path deep | crawl by importance score |
| Coverage | broad, even | narrow, can go deep into one site | focuses effort where it matters |
| Politeness fit | good (spreads across hosts) | **bad** (hammers one host) | good (frontier handles it) |
| Trap risk | lower | **high** (falls into infinite depth) | lower (budgets + scoring) |

**Choose:** **BFS-ish with priority.** Pure DFS is dangerous (hammers a host, falls into traps). Real crawlers use
a **priority frontier** that's broadly breadth-first but crawls important pages sooner. See
[frontier](deep-dives.md#1-the-url-frontier-the-heart-of-the-crawler).

---

## 2. Bloom filter vs exact set for URL dedup

- **Bloom filter:** ~1 GB for 1B URLs, O(1) checks, **no false negatives**; rare **false positive** → occasionally
  skip a new URL. Cheap and scalable.
- **Exact set (KV/DB):** perfect, but ~1 TB+ and a lookup per link → expensive at web scale.

**Choose:** **Bloom filter in front of a sharded KV.** The rare false-positive skip is an acceptable loss for a
crawler; you get billions-scale dedup in gigabytes. → [dedup](deep-dives.md#2-url-deduplication-the-seen-set).

---

## 3. Object storage vs database for pages

- **Object storage (S3):** built for large immutable blobs, effectively infinite, cheap, write-once/sequential-read
  — exactly the page access pattern. Metadata lives in a separate KV.
- **Database:** wrong tool — 100 KB blobs × billions destroy an OLTP DB.

**Choose:** **object storage for bodies + KV for metadata.** Say why: access pattern, not popularity.

---

## 4. Politeness strictness vs throughput

- **Stricter politeness** (≤1/host, long crawl-delay): safest, never blocked, but slower per host.
- **Looser** (parallel connections per host): faster per host, but risks blocks/blacklisting/legal issues.

**Choose:** **strict by default** (≤1 in-flight per host, obey crawl-delay); gain throughput from **breadth across
many hosts**, not from being aggressive on any one. The politeness ceiling is a constraint you design around.

---

## 5. Shard the frontier by host vs by URL-hash

- **By host:** all of a host's URLs + politeness state co-locate → **politeness is a local decision** (no
  distributed lock). This is why it's the standard.
- **By URL-hash:** even load distribution, but a host's URLs scatter → you'd need **distributed per-host rate
  limiting** (contended, complex).

**Choose:** **shard by host.** It makes the hardest problem (distributed politeness) disappear. (Watch for a
mega-host becoming a hot shard → sub-partition it — see [hot shards](../../01-patterns/sharding.md#hot-shards--hot-partitions).)

---

## 6. Recrawl: fixed schedule vs adaptive

- **Fixed timer:** simple, but wastes bandwidth recrawling static pages and misses fast-changing ones.
- **Adaptive (change-rate + conditional GET):** recrawl by estimated volatility; `304` makes checks nearly free.

**Choose:** **adaptive**, using conditional GETs and sitemap `lastmod`. → [freshness](deep-dives.md#7-freshness--recrawl-turning-a-one-shot-into-a-living-index).

---

## 7. Render JavaScript vs fetch plain HTML

- **Plain HTML:** cheap, fast — but misses SPA content.
- **Headless render:** sees everything, but **10–100× cost**.

**Choose:** **plain HTML by default; a targeted render tier** only for sites that need it. Don't render the whole web.

---

## 8. Build vs buy the fetch/render layer

- **Build:** full control, tuned throughput; more ops.
- **Managed (proxies, render services):** faster to stand up, handles IP rotation/blocks; recurring cost + lock-in.

**Choose:** depends on scale and whether anti-bot evasion is needed; at true web scale you build the core and may
buy proxy/render capacity. Name the trade rather than defaulting.

---

## The one-paragraph summary (say this)

> *"It's throughput- and coordination-bound, not latency-bound. The core is a two-level URL frontier — front
> queues for priority, back queues pinned one-per-host for politeness — sharded by host so per-host rate limiting
> is a local decision with no distributed lock. Fetchers are stateless async I/O workers consulting cached DNS and
> robots.txt. I dedup URLs with a Bloom filter over a sharded KV, and content with SimHash for near-duplicates.
> Pages go to object storage (compressed) with metadata in KV. I guard against traps with per-host URL/depth/size
> budgets, and keep the index fresh with adaptive, conditional-GET recrawls. Plain HTML by default; a targeted
> headless-render tier only where JS is required. Throughput scales by adding fetchers and frontier shards;
> politeness is the hard ceiling I design breadth around."*

→ Next: **[Failure Scenarios](failure-scenarios.md)**
