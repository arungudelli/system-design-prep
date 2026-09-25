# Web Crawler

> Design a system like Googlebot: start from a set of seed URLs, fetch pages, extract the links on them, and
> keep following those links — crawling a large portion of the web, politely, at scale, and refreshing content
> over time.

This is a **write/throughput-heavy, coordination-heavy** system — the opposite profile from the read-heavy URL
shortener. The signal is in the **URL frontier** (a priority + politeness queue), **deduplication** (don't fetch
the same URL or store the same content twice), **politeness** (don't hammer a site / obey `robots.txt`),
**distributed coordination** across thousands of fetchers, and **trap avoidance**. It's the system that pulls
together nearly every pattern you've studied.

---

## Why it's a great system to study

- **Uses the whole pattern library:** the frontier is a [queue + workers](01-patterns/queues-workers.md); URL/
  content dedup is [idempotency](01-patterns/idempotency.md) at scale; the frontier and storage are
  [sharded](01-patterns/sharding.md); DNS and `robots.txt` are [cached](01-patterns/caching.md).
- **Politeness = distributed rate limiting** — per-domain, across many machines.
- **Bounded infinite work:** the web is effectively infinite (and adversarial — crawler traps). Prioritization,
  dedup, and limits are the whole game.
- **Throughput at massive scale** with a hard external constraint (you can't out-crawl a site's tolerance).

---

## Read in this order

1. **[Requirements](02-systems/web-crawler/requirements.md)** — problem, clarifying questions, functional + NFRs
2. **[Capacity](02-systems/web-crawler/capacity.md)** — pages/sec, storage, bandwidth, fetcher count
3. **[API & Data Model](02-systems/web-crawler/api-data-model.md)** — internal interfaces + the frontier/dedup/store schemas
4. **[Architecture](02-systems/web-crawler/architecture.md)** — components, the crawl loop, scaling
5. **[Deep Dives](02-systems/web-crawler/deep-dives.md)** — URL frontier, dedup (Bloom + content hash), politeness, DNS, traps, freshness
6. **[Trade-offs](02-systems/web-crawler/tradeoffs.md)** — the decisions, both sides
7. **[Failure Scenarios](02-systems/web-crawler/failure-scenarios.md)** — what breaks and how we contain it
8. **[Interview Questions](02-systems/web-crawler/interview-questions.md)** — attempt first, then reveal
9. **[Cheatsheet](02-systems/web-crawler/cheatsheet.md)** — 1-page revision

---

## The 30-second version (know this cold)

- **Scale target:** ~1B pages/month → ~**400 pages/s** avg (peak ~1–2k/s); ~**60 TB/month** of raw HTML.
- **Core loop:** frontier → fetch → parse → extract links → **dedup URLs** → enqueue new URLs; store content.
- **URL Frontier** = the heart: a set of queues giving **priority** (crawl important pages first) *and*
  **politeness** (one polite connection per host at a time, respect crawl-delay).
- **Dedup URLs:** a **Bloom filter** (+ backing store) of "already seen" URLs to avoid re-enqueueing — memory-efficient at billions of URLs.
- **Dedup content:** hash the page (and near-dup detect with **SimHash/MinHash**) so mirrored/duplicate pages aren't stored/processed twice.
- **Politeness:** obey `robots.txt` (cached per host), cap concurrency + rate **per domain**, identify via User-Agent.
- **DNS is a hidden bottleneck** → cache aggressively, use async resolution.
- **Traps:** infinite calendars, session-id URLs, deep dynamic paths → depth/URL-count limits per domain + trap heuristics.
- **Storage:** raw pages → **object storage (S3)**; URL/frontier metadata → sharded KV; links → for the graph/indexer.
- **Bottleneck order:** network bandwidth & fetcher concurrency → DNS → frontier throughput → storage writes.
