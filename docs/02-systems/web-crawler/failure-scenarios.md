# Web Crawler — Failure Scenarios

> A crawler runs for months against a hostile, unreliable web. The themes: **never lose the frontier**
> (that's the accumulated work), **never crash a worker on one bad page**, **survive traps and slow servers**,
> and **be resumable**.

---

## Fetcher crashes mid-download

- **Impact:** the URLs it had leased from the frontier are in-flight and unfinished.
- **Detect:** lease expiry / missing heartbeat.
- **Contain/recover:** frontier entries are **leased with a timeout** (like a [visibility timeout](../../01-patterns/queues-workers.md#acknowledgment--the-visibility-timeout-the-core-mechanic)); on expiry they become eligible again and another fetcher picks them up.
  A rare **double-fetch is harmless** because dedup/idempotency de-duplicate the result.
- Fetchers are **stateless** → just replace/restart; autoscale the pool.

## Frontier node fails

- **Impact:** the URLs on that shard (a subset of hosts) are temporarily unavailable → those hosts stall.
- **Detect:** shard health check.
- **Contain:** the frontier is **persisted/replicated per shard** → promote a replica; other shards keep crawling
  (blast radius = the hosts on that one shard).
- **Recover:** restore from the durable log/checkpoint; **never** rebuild the frontier by re-crawling the web.
- **This is why the frontier must be durable, not in-memory** — losing it means losing months of discovery.

## Seen-set / Bloom filter lost

- **Impact:** without "seen" state we'd re-enqueue already-crawled URLs → wasted work, not incorrectness.
- **Contain/recover:** **persist/snapshot** the Bloom filter and back it with a sharded KV; on restart, reload the
  snapshot + replay recent additions. A cold seen-set causes a temporary spike of re-fetches (dedup on the KV
  backing still catches most), which is tolerable but wasteful → snapshot regularly.

## A crawler trap (infinite URL space)

- **Impact:** one host generates endless unique URLs (calendar, session ids, faceted nav) → the frontier fills
  with junk from one host; real crawling starves.
- **Detect:** abnormal **URLs-per-host** growth, parameter explosion, very deep paths.
- **Contain:** **per-host URL budget + depth limit + parameter/pattern heuristics + session-id stripping** in
  normalization. Blacklist pathological patterns. (See [traps](deep-dives.md#6-crawler-traps--hostile-content-robustness).)

## A target server is slow / hostile (200 ms → 30 s, or returns garbage)

- **Impact:** fetch threads tie up waiting; oversized/streaming responses exhaust memory.
- **Contain:** **per-request timeout**, **max page size** (stream + abort oversized), **redirect-chain cap**,
  tolerant parsing (bad HTML never crashes a worker). One slow host must not stall others (its back-queue just
  progresses slowly).

## A site rate-limits / blocks us (429 / 403)

- **Impact:** requests to that host fail.
- **Detect:** rising 429/403 per host.
- **Contain:** **exponential backoff per host**, honor `Retry-After`, tighten that host's crawl-delay, and — if
  persistent — **park the host** and alert. Being politer is the fix; never retry-storm a host that's pushing back.

## DNS resolver overload / failure

- **Impact:** fetches can't resolve hosts → throughput collapses.
- **Detect:** DNS latency/error spike.
- **Contain:** **DNS cache** absorbs most lookups; run **redundant caching resolvers**; async resolution with
  timeout+retry; serve **stale cache** entries during resolver outages rather than failing.

## Object storage (page store) throttles or is unavailable

- **Impact:** can't persist fetched pages.
- **Detect:** write errors / S3 throttling.
- **Contain:** **buffer/backpressure** — if storage is slow, slow the fetchers (bounded in-flight); retry writes
  with backoff; batch/multipart uploads. Don't drop fetched content silently; apply backpressure up the pipeline.

## Downstream (indexer) can't keep up

- **Impact:** the crawler produces pages faster than the indexer consumes.
- **Contain:** decouple via a **durable queue/object store** between crawler and indexer → the crawler isn't blocked
  by the indexer, and the indexer catches up at its own rate ([queues & workers](../../01-patterns/queues-workers.md)).

## Full restart / deploy

- **Impact:** everything stops; must resume without re-crawling the web.
- **Contain:** **checkpoint** frontier + seen-set continuously; on restart, reload state and continue. Rolling
  deploys so only a fraction of fetchers restart at once; leases cover the in-flight URLs of restarting workers.

## Whole-region / network-egress failure

- **Impact:** lose crawl capacity / bandwidth in a region.
- **Contain:** multi-region fetcher pools; shards can be re-assigned to healthy regions; the durable frontier means
  no discovered work is lost.

---

## Failure-handling toolkit used here

`leased frontier entries` (redeliver on crash) · `durable/replicated frontier` (never lose work) ·
`Bloom-filter snapshots` · `per-host backoff + Retry-After` · `per-request timeout / max page size / redirect cap`
· `per-host URL & depth budgets` (traps) · `DNS cache + redundant resolvers + stale-serve` ·
`backpressure to storage/indexer` · `checkpoint + resume` · `stateless fetchers` (replace freely).

## Priorities (say this)

> *"My durability budget goes to the **frontier and seen-set** — they're the accumulated crawl state; losing them
> means re-crawling the web, so they're persisted, replicated, and checkpointed. Individual fetches are disposable:
> a crashed fetcher's leased URLs simply get redelivered, and dedup makes the rare double-fetch harmless. And the
> whole system must survive a hostile web — timeouts, size caps, trap budgets, and per-host backoff keep one bad
> site from taking down the crawl."*

→ Next: **[Interview Questions](interview-questions.md)**
