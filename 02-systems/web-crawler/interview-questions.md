# Web Crawler — Interview Questions

> Answer each **out loud first**, then expand the model answer and self-critique against the
> [critique lens](00-framework/staff-level-thinking.md#how-your-answer-gets-graded-the-critique-lens).

---

## Core

**Q1. What's the profile of this system, and how does it differ from a read-heavy service?**
<details><summary>Model answer</summary>

It's **throughput/bandwidth-bound and coordination-heavy**, not read-latency-bound. The work is: fetch at scale,
dedup billions of URLs, stay polite per host, store huge amounts of blob data, and survive a hostile web. So the
design centers on the **frontier**, **dedup**, **politeness**, and **object storage** — not on caches + replicas
for low-latency reads. Naming this profile up front frames every later decision.
</details>

**Q2. Design the URL frontier. How does it give both priority and politeness?**
<details><summary>Model answer</summary>

Two levels: **front queues** by priority (a prioritizer routes URLs to priority levels; higher levels chosen more
often) → **back queues** each pinned to **one host** (only one worker drains a back-queue → ≤1 connection per host),
plus a **heap of per-host next-eligible times** to honor crawl-delay. **Shard the whole frontier by host** so
politeness is a local decision. Persist it (durable). See [frontier](02-systems/web-crawler/deep-dives.md#1-the-url-frontier-the-heart-of-the-crawler).
</details>

**Q3. How do you dedup URLs at billions scale without a terabyte of RAM?**
<details><summary>Model answer</summary>

**Bloom filter** — ~1.2 GB for 1B URLs at ~1% false positive, O(1) checks, **no false negatives**. A false positive
just skips a genuinely new URL occasionally — acceptable for a crawler. Back it with a **sharded KV** (`url_hash →
crawled_at`) for exactness. **Normalize URLs first** or dedup is meaningless. See [dedup](02-systems/web-crawler/deep-dives.md#2-url-deduplication-the-seen-set).
</details>

**Q4. Why store pages in object storage instead of a database?**
<details><summary>Model answer</summary>

Pages are **large immutable blobs** (~100 KB) written once and read sequentially by downstream jobs — the exact
object-storage (S3) access pattern. Billions of 100 KB blobs would destroy an OLTP DB. Bodies → object storage
(compressed); metadata (url_hash, status, etag, content_hash, body pointer) → a KV store. Choose per access pattern.
</details>

---

## Politeness & correctness

**Q5. How do you enforce politeness across thousands of fetcher machines without a distributed lock per host?**
<details><summary>Model answer</summary>

**Shard the frontier by host.** All of a host's URLs funnel through one shard/back-queue drained by one worker →
per-host rate limiting becomes a **local** decision. If a host could be fetched from any machine you'd need a
contended distributed rate-limiter per host; host-sharding makes that problem vanish. Plus cached `robots.txt`
(crawl-delay) and a per-host next-eligible heap.
</details>

**Q6. How do you handle robots.txt efficiently?**
<details><summary>Model answer</summary>

Fetch and parse it **per host**, **cache it with a TTL** (it's consulted on essentially every fetch to that host),
respect `Disallow`/`Crawl-delay`/`noindex`/`nofollow` per policy, and use its sitemap hints. Caching is essential —
without it you'd re-fetch robots.txt constantly. It's a per-host [cache](01-patterns/caching.md) just like DNS.
</details>

**Q7. The same article appears at 500 URLs. How do you avoid storing/indexing it 500 times?**
<details><summary>Model answer</summary>

**Content dedup:** exact duplicates via a content hash (SHA-256 of normalized content); **near-duplicates** (differ
only in ads/timestamps) via **SimHash/MinHash** — similar docs get similar fingerprints, collapse within a small
Hamming distance. Store one canonical copy. See [content dedup](02-systems/web-crawler/deep-dives.md#3-content-deduplication-near-duplicate-detection).
</details>

**Q8. Why is DNS a problem, and what do you do?**
<details><summary>Model answer</summary>

A lookup per fetch adds latency and floods resolvers (400+/s). **Cache resolutions per host** (many URLs share a
host) with TTL, **resolve asynchronously**, and at scale **run your own caching resolvers**. Serve stale entries
during resolver outages rather than failing. DNS is the classic "forgot about it and it bottlenecked" component.
</details>

---

## Robustness / traps

**Q9. A site has an infinite calendar (`?date=…` forever). What happens and how do you prevent it?**
<details><summary>Model answer</summary>

Without guards the frontier fills with endless unique URLs from one host and real crawling starves. Prevent with
**per-host URL budgets, crawl-depth limits, parameter-explosion detection, session-id stripping in normalization**,
and pattern blacklists. Detect via abnormal URLs-per-host growth. See [traps](02-systems/web-crawler/deep-dives.md#6-crawler-traps--hostile-content-robustness).
</details>

**Q10. A target server hangs for 30 seconds or returns a 2 GB page. How do you protect the crawler?**
<details><summary>Model answer</summary>

**Per-request timeout**, **max page size** (stream and abort oversized bodies), **redirect-chain cap**, and
tolerant parsing so malformed HTML never crashes a worker. One slow/hostile host must not stall others — its
back-queue just progresses slowly while other hosts continue.
</details>

**Q11. A site starts returning 429/403. What do you do?**
<details><summary>Model answer</summary>

**Back off per host** (exponential), honor **`Retry-After`**, tighten that host's crawl-delay, and if persistent,
**park the host** and alert. The fix is to be politer, never to retry-storm a host that's already pushing back.
</details>

---

## Scale, freshness, failure

**Q12. A fetcher crashes mid-download. Is anything lost? Could a page be fetched twice?**
<details><summary>Model answer</summary>

Nothing lost: frontier entries are **leased** (visibility-timeout style); on lease expiry the URL is redelivered to
another fetcher. A page **could** be fetched twice (once by the dead worker, once by the new one), but **dedup makes
that harmless** — [idempotency](01-patterns/idempotency.md) at work. Fetchers are stateless → just replace them.
</details>

**Q13. How do you keep the crawl fresh without recrawling everything constantly?**
<details><summary>Model answer</summary>

**Adaptive recrawl** by estimated change rate (news homepage often, static pages rarely) + **conditional GETs**
(`If-Modified-Since`/etag → `304 Not Modified` costs ~no bandwidth) + **sitemap `lastmod`/feeds** to learn what
changed. The frontier's priority + next-eligible time doubles as the recrawl scheduler. See [freshness](02-systems/web-crawler/deep-dives.md#7-freshness--recrawl-turning-a-one-shot-into-a-living-index).
</details>

**Q14. What's the first bottleneck, and how do you scale throughput?**
<details><summary>Model answer</summary>

**Network bandwidth + fetch concurrency** (it's I/O-bound). Scale by adding **stateless async fetchers** (thousands
of concurrent connections each), distributing across network egress/regions. Then DNS (cache), then frontier
(shard by host), then seen-set (Bloom+KV), then storage (object store scales itself). **Politeness is a hard
ceiling per host** → throughput comes from **breadth across many hosts**, not depth on a few.
</details>

**Q15. What changes at 10×? At 100×?**
<details><summary>Model answer</summary>

**10×** (~4k pages/s, multi-Gbps): more fetcher machines + more frontier shards; run own DNS resolvers; watch for
mega-hosts as hot shards (sub-partition). **100×** (approaching real search scale): multi-region fetchers near the
content, a dedicated freshness/priority scheduler, tiered storage with retention, a selective JS-render tier, and
serious bandwidth-cost engineering. The frontier + seen-set stay the coordination backbone.
</details>

**Q16. The frontier node dies. Why is this the scariest failure, and how do you handle it?**
<details><summary>Model answer</summary>

The frontier is **accumulated discovery** — months of "what to crawl." Losing it in-memory means re-crawling the
web. So it must be **durable + replicated per shard**: promote a replica, restore from the log/checkpoint; other
host-shards keep crawling (blast radius = that shard's hosts). This is why the frontier is persisted, not just an
in-RAM queue. See [failures](02-systems/web-crawler/failure-scenarios.md#frontier-node-fails).
</details>

---

## Staff-level curveballs

**Q17. Half the web is JavaScript-rendered SPAs now. Do you render everything?**
<details><summary>Model answer</summary>

**No.** Headless rendering is **10–100× the cost** of a plain fetch. Fetch plain HTML by default and route only
sites known to need JS to a **separate render tier** (detect empty-shell HTML / known SPA hosts). Rendering
everything would blow the bandwidth/compute budget for marginal coverage gain. Name the trade-off explicitly.
</details>

**Q18. How do you make the crawl polite AND high-throughput at the same time — aren't they opposed?**
<details><summary>Model answer</summary>

They're only opposed *per host*. You can't exceed one host's tolerance, so global throughput comes from crawling
**many hosts in parallel** — breadth. The host-sharded frontier + one-connection-per-host back-queues let you be
strictly polite to each host while thousands of hosts are fetched simultaneously. Politeness bounds per-host rate;
parallel breadth delivers aggregate throughput.
</details>

**Q19. How would you prioritize what to crawl first with a limited budget?**
<details><summary>Model answer</summary>

Score URLs by **importance** (inbound links / PageRank-ish signals, domain authority), **freshness/change rate**,
and **purpose fit** (for a topical crawl). Feed the score into the frontier's **front (priority) queues** so
high-value pages are crawled sooner and recrawled more often. With a finite budget you crawl the highest expected
value per byte — coverage isn't uniform.
</details>

**Q20. How do you avoid re-processing content you've already seen after a restart, without a perfect record?**
<details><summary>Model answer</summary>

**Snapshot the seen-set (Bloom filter) + persist the backing KV**; on restart reload the snapshot and replay recent
additions. A cold seen-set causes a temporary re-fetch spike (still caught by the KV backing on write), which is
wasteful but not incorrect — so snapshot frequently. Content dedup (hash/SimHash) also prevents re-*storing*
duplicates even if a URL is re-fetched.
</details>

**Q21. Where does this system use each pattern you've studied?**
<details><summary>Model answer</summary>

**Queue + workers:** the frontier is a durable, host-sharded queue drained by fetcher workers with leases.
**Idempotency:** URL/content dedup makes redelivered/duplicate fetches harmless. **Sharding:** frontier + seen-set
+ metadata sharded (by host / by url-hash). **Caching:** DNS + robots.txt caches. **Object storage:** raw pages.
**Rate limiting:** per-host politeness. **Backpressure:** slow storage/indexer slows fetchers. That composition is
the whole design.
</details>

**Q22. When would you redesign this system?**
<details><summary>Model answer</summary>

When a core assumption changes: the crawl purpose narrows/broadens dramatically (topical vs whole-web), JS-rendered
content becomes the majority (render-first architecture), freshness needs go real-time (push/feed-driven ingestion
instead of polling), or scale jumps 100× (multi-region, tiered storage, dedicated schedulers). Absent those, the
frontier + dedup + politeness backbone holds.
</details>

---

## Self-scoring

Grade against the [15-point critique lens](00-framework/staff-level-thinking.md#how-your-answer-gets-graded-the-critique-lens):
did you frame the throughput/coordination profile, design the frontier for priority+politeness, dedup at scale,
handle traps + hostile servers, make the frontier durable, and connect it to the patterns? Note your weakest area
and drill it.

← Back to **[README](02-systems/web-crawler/README.md)** · Next: **[Cheatsheet](02-systems/web-crawler/cheatsheet.md)**
