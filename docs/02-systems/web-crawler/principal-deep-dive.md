# Web Crawler — Principal-Level Deep Dive (The Stops)

> The **"never be blind"** companion to the base Web Crawler docs. It lays out the crawler as a **staircase of
> stops** — the smallest defensible design first, then each bigger stop with the **exact trigger** that forces it —
> and ends with the **top-stop (max-scale / deliberately over-engineered) architecture** plus a **reconciliation
> table** telling you when each heavy component is overkill and should be **removed**.
>
> Read the base first: [Requirements](requirements.md) · [Architecture](architecture.md)
> · [Deep Dives](deep-dives.md) · [Trade-offs](tradeoffs.md).
>
> **Say this in an interview:** *"I'll start with the smallest crawler that meets the requirement and grow it one
> bottleneck at a time. I know the fully-distributed, multi-region, JS-rendering, ML-prioritized version — but I'd
> only reach for each piece when a specific ceiling forces it, and I'll tell you which ceiling."*

---

## The stops (climb one lever at a time)

| Stop | Scale target | Shape | **Trigger that forces the NEXT stop** |
|---|---|---|---|
| **1 · Single-process** | thousands of pages; a dataset/POC | one process: in-memory frontier + `seen` set, synchronous fetch→parse→store to disk | one machine can't hit the throughput; a crash loses the frontier |
| **2 · Worker pools + durable frontier** | ~10–50 pages/s | fetcher/parser worker pools + **durable queue** frontier + **Bloom-filter** seen-set + **object storage** for pages | frontier contention; DNS latency; duplicate content wasting storage |
| **3 · Sharded + polite + deduped** | ~400+ pages/s (our base target) | frontier **sharded by host** (priority front + one-host-per back queues) + fetchers with host affinity + **DNS/robots caches** + **SimHash** content dedup | bandwidth/geography limits; SPA sites; freshness demands; single-region risk |
| **4 · Global, rendering, adaptive (top stop)** | thousands+ pages/s, continuous, web-scale | **multi-region** fetcher fleets + **JS-render tier** + **ML-driven priority & freshness** scheduler + **tiered storage** + dedicated distributed dedup service | (ceiling of practical single-org crawling) |

> Most interviews want **Stop 3**. Reach into Stop 4 only when the prompt says "like Googlebot / whole-web /
> real-time freshness / render everything." Naming the trigger each time is the signal.

---

## Top-stop architecture (max-scale)

```text
                         Seed & recrawl scheduler (ML: importance + change-rate)
                                        │  prioritized URLs
        ┌──────────────────────── URL FRONTIER (sharded by host, durable, multi-region) ─────────────┐
        │  front queues (priority)  →  back queues (1 host each, politeness) + next-eligible heap      │
        └───────────────────────────────────┬────────────────────────────────────────────────────────┘
             region US ▼           region EU ▼            region APAC ▼   (fetch near the content)
        [ Fetcher fleet ]      [ Fetcher fleet ]      [ Fetcher fleet ]   async I/O, own DNS resolvers, robots cache
             │  plain HTML                                  │
             │  needs JS? ──► [ Headless render farm ] ─────┘   (10–100× cost — only for flagged hosts)
             ▼
        [ Parser / extractor fleet ]
             │  URL dedup → distributed Bloom + sharded KV     content dedup → SimHash service
             ├──► pages → object storage (compressed, tiered hot/cold) + metadata → sharded KV
             ├──► link edges → graph store (offline PageRank-style ranking)
             └──► unseen in-scope URLs → back to FRONTIER
             │
        crawl telemetry ──► Kafka ──► stream processing ──► dashboards (coverage, freshness, trap detection)
```

---

## Deep dives that only matter at the top stop

### Multi-region fetching (Stop 3 → 4 trigger: **bandwidth + geography**)
- **Why:** at thousands of pages/s, single-region **egress bandwidth** saturates and you're far (latency) from much
  of the web. Fetch **from the region nearest the content** → lower latency, spread bandwidth, respect geo/legal.
- **Hard part:** the **frontier and seen-set are global state**. Options: partition hosts to regions (a host is
  crawled by one region → politeness stays local), replicate the seen-set (Bloom snapshots) region-to-region, and
  reconcile discovered URLs asynchronously. Don't try to make the global seen-set strongly consistent — eventual is fine
  (a rare double-fetch is harmless via dedup).
- **Over-engineering watch:** *only when* one region's bandwidth is the ceiling or you must crawl geo-restricted
  content. A single-region fleet handles a surprising amount; **remove** multi-region below that.

### JS-render tier (trigger: **content is client-rendered**)
- **Why:** SPAs return an empty shell to a plain fetch. Rendering needs a **headless browser** → **10–100× CPU/
  memory** per page.
- **Design:** a **separate render farm**; route only **flagged hosts** (detect empty-shell HTML / known SPA
  domains) to it; plain-fetch everything else. Cap render concurrency; it's the most expensive resource.
- **Over-engineering watch:** *only for the subset that needs it.* Rendering the whole web is a budget-killer;
  **remove**/skip rendering for static-HTML sites (still the majority of pages by volume).

### ML-driven priority & freshness (trigger: **finite budget, must crawl the *right* pages**)
- **Why:** you can't crawl everything continuously; spend the budget on **high-value, frequently-changing** pages.
- **Design:** score URLs by importance (link graph / PageRank-ish) + estimated change rate; feed scores into the
  frontier's **priority queues** and the **recrawl scheduler**. Use conditional GETs (`304`) so freshness checks are
  nearly free.
- **Over-engineering watch:** *only at web-scale with a real budget constraint.* At smaller scope, simple BFS +
  fixed recrawl cadence is fine; **remove** the ML scoring.

### Distributed dedup service (trigger: **seen-set + content-hash outgrow a node**)
- **Why:** billions of URLs and content fingerprints exceed one machine.
- **Design:** shard the Bloom filter + backing KV by `hash(url)`; a **SimHash/MinHash service** for near-dup at
  scale; snapshot for restart. See [dedup base](deep-dives.md#2-url-deduplication-the-seen-set).
- **Over-engineering watch:** *only at billions scale.* A single Bloom filter (~1.2 GB/1B) covers a lot; **remove**
  the distributed service below billions.

### Tiered storage (trigger: **continuous crawl → unbounded storage cost**)
- **Why:** a forever-running crawl accumulates PBs. Keep **recent/high-value pages hot**, archive the rest **cold**
  (cheaper object-storage tier), expire by retention policy.
- **Over-engineering watch:** *only when storage cost is a real line item.* Below that, one storage tier + compression
  is simpler; **remove** tiering.

---

## Reconciliation table — what forces each heavy component (and when to remove it)

| Component | Base docs (Stop 3, ~400 pg/s) | Top stop (Stop 4) | Trigger to ADOPT | When it's OVERKILL → remove |
|---|---|---|---|---|
| Multi-region fetchers | single region | fetch near content, global | region bandwidth ceiling / geo content | single region meets throughput |
| JS-render tier | plain HTML only | headless farm for flagged hosts | client-rendered content matters | target sites serve static HTML |
| ML priority/freshness | priority frontier + adaptive recrawl | learned importance + change-rate models | web-scale + finite budget | narrow scope; fixed cadence is fine |
| Distributed dedup service | single Bloom + sharded KV | sharded Bloom + SimHash service | billions of URLs/fingerprints | ≤ ~1B URLs (one Bloom filter fits) |
| Tiered storage | object storage + compression | hot/cold tiers + retention | continuous crawl, PB-scale cost | bounded crawl; one tier is fine |
| Kafka telemetry pipeline | metrics/logs | stream processing for coverage/freshness | ops at scale need real-time crawl analytics | small crawl; simple metrics suffice |

> The discipline: **every row is introduced by a trigger and removed below it.** In an interview, lead with Stop 3,
> then: *"if you push me to whole-web with real-time freshness and SPA coverage, here's Stop 4 and exactly which
> ceiling each piece answers."*

---

## The 60-second Principal summary

> *"The crawler is throughput- and coordination-bound. I start with worker pools around a durable, host-sharded
> frontier (priority front queues + one-host-per back queues so politeness is local), a Bloom-filter seen-set, cached
> DNS/robots, SimHash content dedup, and pages in compressed object storage — that's a few hundred pages/s, robust to
> traps and crashes. If you push to whole-web scale I climb one lever at a time: multi-region fleets when bandwidth/
> geography is the ceiling, a headless-render farm only for the SPA subset (it's 10–100× the cost), ML-driven priority
> and freshness when the budget forces choosing the right pages, a sharded dedup service past a billion URLs, and
> tiered storage when continuous-crawl cost bites. Each piece has a named trigger, and below that trigger I'd remove
> it — reaching for all of it up front would be over-engineering."*

← Back to **[Web Crawler overview](README.md)** · Base **[deep dives](deep-dives.md)** · **[cheatsheet](cheatsheet.md)**
