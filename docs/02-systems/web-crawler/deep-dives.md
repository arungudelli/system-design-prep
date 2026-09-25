# Web Crawler — Deep Dives

The interview lives here. The crawl loop is easy to draw; the signal is the **frontier**, **dedup at scale**,
**politeness**, **DNS**, **traps**, and **freshness**.

---

## 1. The URL Frontier (the heart of the crawler)

The frontier decides **what to crawl next**. It must satisfy two goals that pull against each other:

1. **Priority** — crawl important/fresh pages sooner (a news homepage > a random forum post from 2009).
2. **Politeness** — never hit one host too hard; obey per-host rate/crawl-delay; ideally ≤1 in-flight per host.

A single FIFO queue can't do both. The classic solution (the **Mercator scheme**) is **two stages of queues**:

```text
                incoming URLs
                      │
        ┌─────────────▼─────────────┐
        │   FRONT queues (priority)  │   f queues, one per priority level.
        │   pick by priority         │   A prioritizer routes each URL to a level.
        └─────────────┬─────────────┘
                      │  router moves URLs down, keeping each back-queue non-empty
        ┌─────────────▼─────────────┐
        │   BACK queues (politeness) │   b queues, EACH DEDICATED TO ONE HOST.
        │   one host per queue       │   A worker reads a back-queue → guarantees
        └─────────────┬─────────────┘   only one connection to that host at a time.
                      │
              back-queue selector + a min-heap of "host → next_eligible_time"
                      ▼
                  Fetchers
```

- **Front queues** give **priority**: higher-priority URLs are chosen more often.
- **Back queues** give **politeness**: each back-queue maps to exactly **one host**, and only one worker drains it
  at a time → you never open two simultaneous connections to the same host. A **heap keyed by
  `next_eligible_time`** enforces crawl-delay (a host isn't fetched again until its delay elapses).
- **Sharding:** the whole frontier is **sharded by host** across machines (all of a host's URLs + its politeness
  state live on one shard) → no cross-machine coordination for politeness. See [sharding](../../01-patterns/sharding.md).
- **Durability:** the frontier is **disk-backed / persisted** (it can hold billions of URLs and must survive
  crashes). It behaves like a [durable queue + workers](../../01-patterns/queues-workers.md) with leases so a crashed
  fetcher's in-flight URLs are redelivered.

> **Interview line:** *"I'd use a two-level frontier: front queues for priority, back queues each pinned to a
> single host for politeness, with a heap of per-host next-eligible times to honor crawl-delay. I shard the whole
> frontier by host so politeness state is local — no distributed locking to keep one connection per host."*

---

## 2. URL deduplication (the "seen" set)

We generate ~10 links/page, mostly already-seen. We must cheaply answer **"have I already seen this URL?"** for
billions of URLs — too many to hold as full strings in memory (see [capacity](capacity.md#frontier--seen-set-the-interesting-one)).

**Bloom filter** — a probabilistic membership structure:
- ~**10 bits/element → ~1% false positive**, so 1B URLs ≈ **1.2 GB** (vs ~1 TB for raw URLs).
- **No false negatives** — if it says "not seen," it's definitely new. **False positives** (says "seen" when new)
  just mean we **skip a genuinely new URL occasionally** — an acceptable, tunable loss for a crawler.
- Back it with a **sharded KV store** (`url_hash → crawled_at`) for exactness/metadata; the Bloom filter is the
  fast front-line check that keeps 99% of lookups out of the KV store.

**Normalization first** (see [API & data model](api-data-model.md#url-normalization-dont-skip--its-a-dedup-correctness-issue)) — dedup is only as good as your canonical form.

> **Interview line:** *"A Bloom filter gives me billions-scale 'seen' checks in ~1 GB with no false negatives.
> The only downside is a rare false positive, which just means we occasionally skip a new URL — fine for a
> crawler. I back it with a sharded KV store for exactness."*

---

## 3. Content deduplication (near-duplicate detection)

The same content appears at many URLs (mirrors, print/mobile versions, session-id variants, syndicated articles).
We don't want to store/index the same page many times.

- **Exact duplicates:** hash the normalized content (e.g. SHA-256) → if the hash is seen, skip. Cheap, catches
  byte-identical pages.
- **Near-duplicates:** small differences (ads, timestamps) defeat exact hashing. Use **SimHash** (or MinHash/
  shingling): compute a fingerprint where **similar documents get similar fingerprints**, and treat pages within a
  small Hamming distance as duplicates. This is what real search crawlers use to collapse near-dups.

> **Interview line:** *"Exact dup = content hash. Near-dup = SimHash, so pages that differ only in ads or a
> timestamp collapse to one — otherwise the index fills with the same article a thousand times."*

---

## 4. Politeness (distributed rate limiting + robots)

Being impolite gets you rate-limited, blocked, or blacklisted. Politeness has three parts:

1. **`robots.txt`** — fetch and parse per host (which paths are allowed, `Crawl-delay`, sitemap hints). **Cache it
   per host with a TTL** (it's read on essentially every fetch for that host). Respect `Disallow`, `noindex`,
   `nofollow` per policy.
2. **Per-host concurrency + rate** — default **≤1 in-flight request per host** and honor `Crawl-delay`. Enforced
   naturally by the **one-host-per-back-queue** design (§1) — no distributed lock needed because each host lives on
   one frontier shard. This is [rate limiting](../../01-patterns/rate-limiting.md) *(coming)* specialized per domain.
3. **Identify yourself** — a descriptive `User-Agent` with a contact URL, so site owners can reach you.

**Why the frontier design solves distributed politeness elegantly:** if a host could be fetched from many machines,
you'd need a distributed lock/rate-limiter per host (expensive, contended). By **sharding the frontier by host**,
all of a host's URLs funnel through one shard/worker → politeness is a **local** decision. That's the insight.

---

## 5. DNS (the hidden bottleneck)

Every fetch needs `host → IP`. Naively that's a DNS lookup per fetch, adding latency and hammering resolvers.
- **Cache** resolutions per host (many URLs share a host) with a TTL → most fetches skip DNS entirely.
- **Resolve asynchronously** so DNS latency doesn't stall fetch threads.
- **Run your own caching resolvers** at scale to avoid overloading upstream DNS and to control TTLs.
- DNS can be slow/variable → treat it as an I/O step with its own timeout + retry.

> **Interview line:** *"DNS is easy to forget and it bites you — one lookup per fetch adds latency and floods the
> resolver. I cache per host and resolve async; at scale I'd run caching resolvers."*

---

## 6. Crawler traps & hostile content (robustness)

The web is adversarial. Guard against:
- **Infinite spaces:** calendars (`?date=…` forever), faceted navigation, session-id URLs that generate endless
  unique URLs. → **Limit URLs-per-host** and **crawl depth**; detect **parameter explosion**; strip session ids
  in normalization.
- **Spider traps / link farms:** pages engineered to trap crawlers. → per-host URL budgets, depth limits, and
  detecting abnormal fan-out.
- **Huge/slow responses:** cap **max page size** and **per-request timeout**; stream and abort oversized bodies.
- **Redirect loops:** cap redirect chains.
- **Malformed HTML:** tolerant parsing; never let one bad page crash a worker.
- **Duplicate-generating URLs:** normalization + content dedup catch most.

> **Interview line:** *"I'd cap URLs-per-host, crawl depth, page size, and redirect depth, and strip session ids
> in normalization — otherwise a single calendar or session-id site becomes an infinite crawl."*

---

## 7. Freshness & recrawl (turning a one-shot into a living index)

Content changes; a stale index is a bad index. But recrawling everything constantly is wasteful.
- **Adaptive recrawl:** recrawl frequently-changing, high-value pages (news homepage) often; static pages rarely.
  Estimate change rate from history (`last_modified`, observed diffs).
- **Conditional GET:** send `If-Modified-Since` / `If-None-Match` (etag); a **`304 Not Modified`** costs almost no
  bandwidth → cheap freshness checks.
- **Priority scheduling:** the frontier's priority + `next_eligible_at` doubles as the recrawl scheduler.
- **Sitemaps / change feeds:** use `sitemap.xml` `lastmod` and RSS/push feeds to learn what changed without polling.

> **Interview line:** *"I'd recrawl by estimated change rate, not on a fixed timer, and use conditional GETs so an
> unchanged page returns 304 and costs almost nothing. High-value volatile pages get short intervals; static pages
> get long ones."*

---

## 8. Distributed coordination & fault tolerance

- **Stateless workers around sharded state:** fetchers/parsers hold no durable state → any can do any work;
  scale/replace freely.
- **Leased frontier entries:** a URL handed to a fetcher is **leased** (like a [visibility timeout](../../01-patterns/queues-workers.md#acknowledgment--the-visibility-timeout-the-core-mechanic)); if the fetcher dies, the lease
  expires and the URL is re-fetched. Dedup makes the rare double-fetch harmless (**idempotency**).
- **Checkpointing:** persist frontier + seen-set so a full restart resumes rather than re-crawling the web.
- **Shard rebalancing:** adding capacity re-assigns host-shards; use consistent hashing / pre-sharding so it moves
  little state (see [sharding](../../01-patterns/sharding.md#resharding--rebalancing-the-operational-nightmare)).

---

## 9. JavaScript rendering (the expensive extension)

Many modern sites render content client-side (SPAs) → plain HTML fetch sees an empty shell.
- **Rendering** needs a **headless browser** (e.g. Chromium) per page → **10–100× the CPU/memory** and cost of a
  plain fetch.
- **Do it selectively:** a separate **render tier** only for hosts/pages known to need JS; plain fetch for the rest.
- Big cost/throughput trade-off → **defer unless the crawl purpose requires it**, and even then, target it.

> **Interview line:** *"JS rendering is 10–100× more expensive, so I wouldn't render everything. I'd detect
> which sites need it and route only those to a headless-browser render tier, fetching plain HTML for the rest."*

→ Next: **[Trade-offs](tradeoffs.md)**
