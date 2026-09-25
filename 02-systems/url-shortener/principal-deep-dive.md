# URL Shortener — Principal-Level Deep Dive (High-Scale)

> This is the **exhaustive, production-grade** companion to the base URL Shortener docs, written at the scale where
> the heavy distributed-systems machinery (ZooKeeper key coordination, a Kafka→Flink→ClickHouse analytics pipeline,
> consistent-hashing sharding, anycast/CDN) is genuinely justified — **10B redirects/month**, **99.999%**,
> **<20 ms** redirects.
>
> The base docs deliberately *start simple* and argue against over-engineering at 100M/month. This doc is the
> **"what changes when the numbers get serious"** view. Read the base first: [Requirements](02-systems/url-shortener/requirements.md)
> · [Capacity](02-systems/url-shortener/capacity.md) · [Architecture](02-systems/url-shortener/architecture.md)
> · [Deep Dives](02-systems/url-shortener/deep-dives.md) · [Trade-offs](02-systems/url-shortener/tradeoffs.md).
>
> **Staff-level framing to say out loud:** *"I'd only introduce ZooKeeper, a streaming analytics pipeline, and
> sharding once the scale and SLA justify them. Below that, they're operational cost with no benefit. Here the
> numbers — 10B reads/month at 99.999% and sub-20ms — justify each one, and I'll show the specific bottleneck that
> forces it."*

---

## 1. Requirements & System Scope

### Functional
- **Shorten** a long URL → system-generated 7-char Base62 code, or a **custom alias**.
- **Redirect** `GET /{code}` → original URL with minimal latency.
- **Expiration** dates for links (custom + default TTL policy).
- **Real-time click analytics** — counts, geo, referrer, device, time series.

### Non-Functional (the tight targets that drive the architecture)
| NFR | Target | Architectural consequence |
|---|---|---|
| **Availability** | **99.999%** (~5 min/year down) | Multi-region active-active for the read path; no single-region SPOF; the redirect path must survive an entire region loss. |
| **Redirect latency** | **p99 < 20 ms** (server-side) | Edge/CDN resolution + in-memory cache; the hot path touches **no disk and no cross-region call**. |
| **Scale** | **100:1 read:write**, 10B reads/mo | Read path is the whole game: CDN → cache → replicas. |
| **Durability** | Never lose a mapping | Replicated, quorum-written store; codes immutable. |
| **Consistency** | Eventual for reads (codes immutable); **strong** for code/alias uniqueness | Immutable mapping → cache/replicate freely; uniqueness enforced at write. |
| **Security** | Anti-abuse, rate limiting, spam/phishing prevention | Gateway rate limiting + safe-browsing checks + abuse detection. |

### Out of scope
User auth UI, billing. We focus on the **system architecture** (the API assumes an authenticated principal is supplied).

> **Why 99.999% forces multi-region:** 99.99% (~52 min/yr) is achievable single-region multi-AZ. The **fifth nine**
> (~5 min/yr) means you cannot tolerate a full-region outage on the read path → **active-active multi-region** with
> DNS/anycast failover. This is the single biggest architectural jump from the base design.

---

## 2. Back-of-the-Envelope Capacity Estimations

### Traffic
```text
Writes: 100,000,000 / month ÷ 2,592,000 s (30-day month)  ≈ 38.6  → ~40 writes/sec (avg)
Reads : 10,000,000,000 / month ÷ 2,592,000 s              ≈ 3,858 → ~3,900 reads/sec (avg)
Ratio : 10B / 100M = 100:1  ✓

Peak (2× average):
  Write peak ≈ 80 writes/sec
  Read  peak ≈ ~7,800 reads/sec
```
→ *So what?* **Writes (~80/s peak) are trivial** — a single primary handles them; **reads (~7.8k/s peak)** are the
target of the whole design. Note: even at 10B/month, writes don't force sharding — reads force **caching + CDN**.

### Storage (payload → 5- and 10-year totals)
```text
Per-record payload:
  short_code (7 B) + long_url (~200 B avg) + user_id (16 B) + created_at (8 B)
  + expires_at (8 B) + metadata/overhead (~60 B)                         ≈ 300–500 B
  Use ~500 B/record (conservative, includes index overhead).

Records/year = 100M/month × 12 = 1.2B/year
  5-year records  = 6.0B   → 6.0B × 500 B  ≈ 3.0 TB
  10-year records = 12.0B  → 12.0B × 500 B ≈ 6.0 TB

With replication (×3) + secondary indexes (~×1.5):
  5-year effective  ≈ 3.0 TB × 4.5 ≈ ~13.5 TB
  10-year effective ≈ 6.0 TB × 4.5 ≈ ~27 TB
```
→ *So what?* Even at 10 years, raw data is **single-digit TB**; with replication + indexes it's tens of TB. This
**fits a small sharded cluster comfortably** — storage is *not* the driver; it's read throughput + availability.
(Click-analytics data is separate and **dwarfs** this — see §5C.)

### Bandwidth
```text
Ingress (writes): 80 writes/s × 500 B                 ≈ 40 KB/s   (negligible)
Egress  (reads) : redirect response ≈ 302 + Location header ≈ ~500 B
                  7,800 reads/s × 500 B                ≈ 3.9 MB/s ≈ ~31 Mbps
```
→ *So what?* Bandwidth is **modest** (a 302 is tiny). This is why a URL shortener is **latency-bound, not
bandwidth-bound** — contrast a video platform. Most of this egress is absorbed at the **CDN/edge**, so origin
bandwidth is even smaller.

### Memory / Cache (80/20 rule)
```text
Daily reads = 10B / 30 ≈ 333M reads/day
Cache the 20% of traffic that drives ~80% of reads → ~the hot working set.

Cache key = short_code → long_url ≈ ~100 B/entry (compact).
Estimate hot distinct entries ~ 100M → 100M × 100 B ≈ 10 GB.
If we also cache richer records (~500 B) for a smaller hot set (~20M): 20M × 500 B ≈ 10 GB.

Target: ~10–20 GB of Redis holds the hot set → 90%+ hit ratio.
```
→ *So what?* **~10–20 GB of cache** delivers a 90%+ hit ratio, dropping origin read QPS from ~7.8k/s to <1k/s. With
a **CDN tier in front**, origin sees even less. Cache is the highest-leverage component — see [caching](01-patterns/caching.md).

---

## 3. API & Data Model

### REST endpoints

**Create**
```text
POST /api/v1/shorten
Authorization: Bearer <token>
Idempotency-Key: <uuid>          # retry-safe: same request → same code
{
  "long_url": "https://example.com/very/long/path?x=1",
  "custom_alias": "promo-2026",  # optional
  "expires_at": "2027-01-01T00:00:00Z"  # optional
}
201 Created
{ "short_url": "https://sho.rt/aB3x9Zq", "short_code": "aB3x9Zq",
  "long_url": "https://example.com/...", "expires_at": "2027-01-01T00:00:00Z" }

400 Bad Request   — malformed/unsafe URL
409 Conflict      — custom_alias already taken
429 Too Many Requests — rate limited (+ Retry-After)
```

**Redirect**
```text
GET /{short_code}
302 Found
Location: https://example.com/very/long/path?x=1
Cache-Control: private, max-age=0    # keep analytics + control (see §5B)

404 Not Found   — unknown code
410 Gone        — expired
```

**Delete**
```text
DELETE /api/v1/shorten/{short_code}
Authorization: Bearer <token>       # must own the code
204 No Content
403 Forbidden   — not owner
404 Not Found
```

### Schema

**`urls`**
```text
short_code    VARCHAR(8)  PRIMARY KEY / PARTITION KEY   -- the lookup + shard key
long_url      TEXT NOT NULL
user_id       BIGINT
created_at     TIMESTAMP
expires_at     TIMESTAMP NULL                            -- null = never
INDEX (user_id)                                          -- list a user's links
INDEX (expires_at)                                       -- background sweeper (or store-native TTL)
```

**`click_analytics`** (append-only, time-series; separate store — see §5C)
```text
event_id      UUID
short_code    VARCHAR(8)      -- partition/cluster key with time
ts            TIMESTAMP
country, region, city         -- geo from IP
referrer, user_agent, device
INDEX/PARTITION: (short_code, ts)   -- "clicks for code X over time"
```

### SQL vs NoSQL — concrete justification

**Access patterns (dominant first):**
1. `short_code → long_url` point lookup, ~7.8k/s peak, <20 ms → **keyed O(1) read**.
2. `create` ~80/s → insert + uniqueness on `short_code`/`custom_alias`.
3. `analytics writes` → massive append volume (see §5C), queried by `(code, time)`.

**Choice:**
- **Mapping store → NoSQL wide-column / KV (DynamoDB or Cassandra).** Rationale: the dominant op is a **partition-key
  point lookup at scale**; no joins; we need **horizontal scale + multi-region replication + high availability**
  (99.999%). DynamoDB global tables / Cassandra multi-DC give **active-active multi-region reads** almost for free —
  exactly the fifth-nine requirement. Consistent-hashing partitioning is built in (§5D).
- **Why not PostgreSQL as the mapping store?** A single Postgres primary is a **regional SPOF** and a write/HA
  ceiling; getting to 99.999% multi-region active-active with Postgres is far more operational work (multi-master is
  painful). At this SLA + scale, a natively-distributed KV wins. *(At 100M-read/month scale, Postgres+cache+replicas
  is simpler and correct — see the base [trade-offs](02-systems/url-shortener/tradeoffs.md#2-sql-vs-nosql). The SLA is what tips it.)*
- **Analytics → columnar time-series (ClickHouse)**, not the mapping store — different access pattern (§5C).
- **PostgreSQL still useful** for low-volume relational metadata (accounts, ownership) if desired — a small
  supporting store, not the hot path.

> **Interview line:** *"The mapping is a keyed point-lookup that must be globally available at five nines, so I use a
> natively-distributed KV with multi-region replication (DynamoDB global tables / Cassandra multi-DC). Postgres would
> be a regional SPOF and a multi-master headache at this SLA. Analytics is a separate columnar time-series store
> because its access pattern is aggregation over time, not point lookups."*

---

## 4. High-Level System Architecture

```text
                              ┌──────────────────────────── READ PATH (99.999%, <20ms) ───────────────────────────┐
   Client
     │  DNS
     ▼
 Anycast DNS  ──►  nearest region / healthy region (GeoDNS + health-based failover)
     │
     ▼
   CDN / Edge  ──(edge cache popular codes; absorbs viral hot keys)──►  hit → 302 at the edge
     │ miss
     ▼
 Regional API Gateway / L7 LB  ──(TLS, WAF, rate limiting: token bucket)──►
     │
     ▼
 Redirect Service (stateless, autoscaled)
     │        │
  cache?      └── emit click event (fire-and-forget, off critical path) ──► Kafka ─┐
     │ hit → 302                                                                    │
     │ miss                                                                         │
     ▼                                                                              │
 Distributed KV (DynamoDB global table / Cassandra multi-DC)                        │
   - consistent-hashing partitioned by short_code                                  │
   - multi-region replicas; reads served locally                                   │
                                                                                    │
   ┌───────────────────────── WRITE PATH ──────────────────────────┐               │
   Client → API GW → Shorten Service                                │               │
                        │ get next code range from ▼                │               │
                   Key Generation Service (KGS)                     │               │
                        │ coordinated by ▼                          │               │
                   Apache ZooKeeper (leases disjoint key ranges)    │               │
                        │ INSERT mapping (+ write-through cache)     │               │
                        ▼                                           │               │
                   Distributed KV (primary region, replicated out)  │               │
   └────────────────────────────────────────────────────────────────┘             │
                                                                                    ▼
   ┌────────────────────── ASYNC ANALYTICS PATH ─────────────────────────────────────────────┐
   Kafka / Kinesis (click events)  ──►  Flink / Spark Streaming (enrich: geo, dedup, aggregate)
        ──►  ClickHouse (raw + rollups, time-series)   ──►  Dashboards / Analytics API
        └──► (optional) cold archive to object storage (S3) for replay
   └──────────────────────────────────────────────────────────────────────────────────────────┘
```

**Component roles:**
- **Anycast DNS + CDN/Edge** — route users to the nearest healthy region and resolve **popular codes at the edge**,
  giving <20 ms and absorbing viral hot keys before they reach origin. Health-based DNS failover delivers the fifth nine.
- **API Gateway / L7 LB** — TLS termination, **WAF**, and **rate limiting (token bucket)** to block abusive bots.
- **Redirect Service** — stateless; cache-first lookup; emits click events async; returns 302.
- **Distributed KV** — the mapping store, consistent-hashing partitioned, multi-region replicated (reads local).
- **KGS + ZooKeeper** — collision-free key allocation (see §5A).
- **Kafka → Flink → ClickHouse** — the decoupled analytics pipeline (see §5C).

---

## 5. Deep-Dive Components & Critical Trade-offs

### A. Key Generation Service (KGS) & Race-Condition Prevention

**Hash-on-the-fly vs pre-generation:**
- **MD5/SHA-256 truncation:** `code = base62(hash(url))[:7]`. Truncating a 128/256-bit hash to 42 bits (7×base62)
  makes **collisions inevitable** (birthday paradox). Each create must then **read the DB to check** for a collision
  and **retry** on hit — a read-before-write on *every* create, causing **latency spikes** and DB load under
  concurrency, plus the same URL maps to the same code (bad for per-link analytics).
- **Base62 of a counter (pre-generated):** each integer → a unique 7-char code by construction → **zero collisions,
  no collision-check read**. This is why we pre-generate.

**KGS architecture (offline pre-computation):**
- A dedicated **Key Generation Service** owns a monotonic counter and hands out **key ranges** (blocks) rather than
  individual keys. It can pre-compute/verify Base62 codes offline and keep a buffer of ready ranges.
- App (Shorten) servers **lease a block** (e.g. 100,000 codes), mint codes from it **in memory**, and lease the next
  block before exhausting the current one → **no per-create coordination, no DB lock**.

**Distributed coordination with Apache ZooKeeper (zero collisions, no DB locks):**
```text
ZooKeeper holds the authoritative "next free range" counter.
  Server A ← range [1        .. 100,000]
  Server B ← range [100,001  .. 200,000]
  Server C ← range [200,001  .. 300,000]
Each server increments within its own in-memory range → disjoint by construction → NO collisions.
Range hand-out is an atomic, sequenced operation in ZooKeeper (a small, strongly-consistent write).
```
- ZooKeeper's **atomic, linearizable** updates guarantee two servers never get the same range — this replaces
  per-write DB locking (which would serialize all creates and destroy write throughput).
- **Crash mid-range:** a server that dies "loses" its unused block → we **skip those codes** (space is 3.5T; codes
  are cheap). **Never reuse** a range — reuse could repoint a live short link.
- **Guessability:** sequential codes are enumerable; if privacy matters, pass the counter through a **reversible
  permutation** (Feistel / multiply-XOR mod 62⁷) → still unique, non-sequential.

> **Interview line:** *"I pre-generate Base62 codes from a counter, not hash-truncation, to avoid a collision-check
> read on every write. A KGS leases disjoint key ranges to app servers, and ZooKeeper's linearizable counter
> guarantees the ranges never overlap — so I get zero collisions with no per-write DB lock. A crashed server just
> forfeits its block; I never reuse codes."*

See also the base [key-generation deep dive](02-systems/url-shortener/deep-dives.md#1-unique-short-code-generation-the-core-sub-problem).

### B. Redirection: HTTP 301 vs 302

| | **301 Moved Permanently** | **302 Found (temporary)** |
|---|---|---|
| Browser/CDN caches the mapping | **Yes** — future clicks skip origin | **No** — every click reaches us |
| Per-click analytics | ❌ lost after first cache | ✅ every click counted |
| Change/expire/disable link later | ❌ clients cached it | ✅ full control |
| Origin/server load | **Lower** (offloaded) | Higher |
| Latency | Lower (client cache) | Slightly higher |

**Business-critical impact:** analytics is a core feature here, so we **default to 302** (+ `Cache-Control:
private, max-age=0`) so every click flows through us and we can expire/disable links. We'd choose **301** only for
links where offload matters more than tracking and the target is truly permanent. This is a
control-and-analytics vs offload-and-latency trade — not a right/wrong. (We still get low latency from **our own**
edge/cache, without giving up tracking.)

### C. Asynchronous Analytics & Telemetry Pipeline

**Why not synchronous:** writing a click metric **inline** in the redirect would add DB/write latency to a path with
a **<20 ms budget** and **~7.8k/s** load — and a spike (viral link) would hammer/crash the analytics DB and take
down redirects with it. Analytics must be **fully decoupled** from the redirect's critical path.

**Decoupled pipeline:**
```text
Redirect Service  ──fire-and-forget──►  Kafka / AWS Kinesis   (durable, buffers bursts, replayable)
        │  (returns 302 immediately)          │
        ▼                                      ▼
                              Flink / Spark Streaming
                                - enrich: IP → geo, parse UA/device
                                - dedup (idempotency on event_id)
                                - windowed aggregation (per-code counts, per-minute rollups)
                                      │
                                      ▼
                              ClickHouse (columnar time-series)
                                - raw events (TTL'd) + pre-aggregated rollups
                                - fast GROUP BY code/time/geo for dashboards
                                      │
                                      ▼
                              Analytics API / Dashboards
                              (+ optional cold archive to S3 for replay/reprocessing)
```
- **Kafka/Kinesis:** durable buffer that absorbs viral bursts and decouples producers from consumers; **replayable**
  (reprocess history, add a new consumer). At-least-once → Flink dedups on `event_id`.
- **Flink/Spark Streaming:** stateful stream processing — geo enrichment, sessionization, **windowed rollups**
  (e.g. clicks/min per code) so dashboards read pre-aggregated data, not raw.
- **ClickHouse:** columnar store built for **analytical aggregation over time** (counts, top referrers, geo
  breakdowns) at massive ingest — the right tool vs a row store. Cassandra is an alternative for raw time-series if
  you prefer; ClickHouse wins for ad-hoc aggregation.
- **Volume reality:** 10B clicks/month × ~200 B ≈ **~2 TB/month raw** — *far* bigger than the mapping store, and
  exactly why it lives in its own pipeline with TTL/rollups/archival.

> **Interview line:** *"The redirect is on a 20 ms budget, so click tracking is fire-and-forget into Kafka and never
> touches the redirect's critical path. Flink enriches and rolls up into ClickHouse for fast time-series queries. A
> viral link becomes queue depth we drain, not redirect latency — and Kafka's replayability lets us reprocess or add
> consumers later."*

### D. Database Sharding & Scaling (Consistent Hashing)

Even though **storage is small**, sharding buys **horizontal read/write throughput, blast-radius isolation, and
per-region distribution** at this scale. (Honest note: at 100M-read/month you wouldn't shard — see base
[Q15](02-systems/url-shortener/interview-questions.md); at 10B/month + five-nines it's reasonable, and the KV store
does it natively.)

**Consistent hashing:**
```text
Hash(short_code) onto a ring; each node owns an arc. Adding/removing a node moves only ~1/N of keys
(not everything, as `hash % N` would). Virtual nodes (many small arcs per physical node) smooth distribution.
```
See the full mechanics in [sharding](01-patterns/sharding.md#3-consistent-hashing-the-fix-for-adding-a-node-remaps-everything).

**Shard key — `short_code` (not `user_id`):**
- The **dominant query is the redirect: point lookup by `short_code`.** Sharding by `short_code` keeps every redirect
  a **single-shard** read — no scatter-gather on the hot path.
- **Why not `user_id`?** Redirects don't know the user; a `user_id` shard key would scatter redirect lookups across
  shards. `user_id` is only good for "list my links" (a rare query) → serve that with a **secondary index** instead.

**Mitigating hot keys / virtual-node imbalance:**
- A **viral code** = a hot partition (request-rate, not size). Mitigate above the DB: **CDN/edge** resolves it,
  **local in-process cache** + distributed cache absorb it, and **key replication** spreads a single hot key
  (`code#1..code#N`). The DB rarely sees the viral key at all.
- **Virtual nodes** even out ring imbalance; **pre-sharding** (many logical shards) makes rebalancing move whole
  virtual shards rather than rehashing keys.

### E. Security, Rate Limiting & Expired-URL Cleanup

**Anti-abuse & rate limiting (at the API Gateway):**
- **Token bucket** per API key/IP on `POST /shorten` — allows natural bursts up to bucket size while capping the
  average rate; refill rate = sustained allowance. **Leaky bucket** where you want a *smooth constant* processing
  rate instead of bursts. Enforced at the gateway/edge so abusive traffic never reaches app servers; distributed
  correctness via **atomic Redis (INCR/Lua)**. See [rate limiting](01-patterns/rate-limiting.md).
- **Layered scopes:** per-IP (unauthenticated abuse), per-API-key (quota), global (protect backend).
- Return **429 + Retry-After** so good clients back off.

**Spam / phishing / malware prevention (see also §6.3):**
- **Safe-browsing check** on create (Google Safe Browsing / domain reputation / blocklists); reject or quarantine.
- **Scheme validation** (http/https only), open-redirect hygiene, optional **interstitial warning** for untrusted targets.
- **Async re-scan** (a URL can turn malicious later) via the pipeline; disable a code by flipping status (302 lets us).

**Two-tier expired-link cleanup (garbage collection):**
1. **Lazy deletion (on access):** on redirect, if `expires_at < now` → return **410 Gone** and delete/tombstone
   lazily. Zero background cost for links that are still being hit; the check is a field comparison already in the read.
2. **Background sweeping (off-peak batch):** a **low-priority async job** scans the `expires_at` index (or uses the
   store's native **TTL** — DynamoDB TTL / Cassandra TTL / Redis TTL) to reclaim links that expired but are never
   accessed again (lazy deletion alone would never remove those). Run during off-peak to avoid competing with the
   read path; throttle it so it never impacts redirects.
- **Why both:** lazy handles hot links cheaply; sweeping/TTL handles the long cold tail so storage + key space don't
  leak. See base [expiry deep dive](02-systems/url-shortener/deep-dives.md#6-expiry--ttl).

---

## 6. Edge Cases & Resilience Strategy

### 1. KGS failure (ZooKeeper leader down / KGS instance crash mid-range)
- **KGS instance crashes mid-range:** it keeps a **leased in-memory block**; if it dies, the **unused portion is
  forfeited** (codes skipped — cheap; never reused). App servers that had leased their own blocks **keep minting**
  from them, so creation continues even while KGS is briefly unavailable. **Lease generous blocks + refill early
  (at ~20% remaining)** so a short KGS outage is invisible.
- **ZooKeeper leader fails:** ZooKeeper runs as an **ensemble (3–5 nodes)** and elects a new leader via
  **ZAB/quorum in seconds**; the counter state is replicated/persisted → **no ranges lost or duplicated**. During the
  brief election, app servers mint from their existing blocks → no user-visible impact.
- **Net:** the durable ZooKeeper counter + pre-leased blocks make key allocation resilient to both failure modes;
  worst case is a few thousand skipped codes.

### 2. Viral URLs (hot-key problem) & cache-stampede prevention
- **CDN/edge** resolves the viral code near users → most traffic never reaches origin.
- **Local in-process cache** on each redirect server + distributed cache → the origin serves the hot key from memory;
  **replicate the hot key** across cache nodes to spread a single-node bottleneck.
- **Stampede prevention** when the hot key expires/cache is cold: **single-flight / request coalescing** (one loader
  repopulates; others wait), **jittered TTLs** (avoid synchronized expiry), and **pre-warming** the top-N.
- **Negative caching + Bloom filter** so a flood of unknown codes (penetration) can't hammer the DB.
- See [caching failure modes](01-patterns/caching.md#the-three-classic-cache-failure-modes-name-these--theyre-common-follow-ups).

### 3. Malicious link insertion (phishing / malware)
- **On create:** validate scheme; run the target through **safe-browsing / reputation / blocklists**; reject `400`
  or quarantine suspicious URLs; rate-limit + bot-detect at the gateway to stop bulk abuse.
- **Continuously:** URLs can turn malicious after creation → **async re-scan** via the analytics/scanning pipeline;
  **disable** a code instantly by flipping status (works because we serve **302**, so we retain control).
- **Defense in depth:** WAF, per-account abuse scoring, interstitial warnings for low-reputation targets, and an
  **abuse-report** endpoint feeding the blocklist.

---

## Reconciling with the base docs (staff judgment)

| Decision | Base docs (100M reads/mo) | This doc (10B reads/mo, 99.999%, <20ms) | Why it changed |
|---|---|---|---|
| Store | Postgres + replicas + cache is fine | **Distributed KV, multi-region** | Five-nines needs active-active; no regional SPOF |
| KGS coord | "lease ranges" (generic) | **ZooKeeper ensemble** named | Explicit linearizable coordination at scale |
| Analytics | "queue + async consumers" | **Kafka → Flink → ClickHouse** | 2 TB/month; real-time rollups |
| Sharding | **Don't shard** (Q15) | **Consistent hashing by short_code** | Throughput + multi-region + blast radius |
| Front door | starts at LB | **Anycast DNS + CDN/edge** | <20 ms + five-nines + hot-key absorption |
| Availability | 99.99% single-region multi-AZ | **99.999% multi-region active-active** | The extra nine |

> The point isn't that the base design was wrong — it's that **each piece of heavy machinery is introduced by a
> specific bottleneck/SLA**, exactly as a Principal engineer should justify it. If the interviewer sets 100M/month
> and 99.9%, you'd *remove* most of this and say so.

---

## The 60-second Principal summary

> *"Read-heavy 100:1 at 10B reads/month, five-nines, sub-20ms. The read path is anycast-DNS → CDN/edge → regional
> stateless redirect service → in-memory cache → multi-region distributed KV, so a redirect is a single-shard,
> in-memory, in-region operation that survives a whole-region loss. Codes come from a counter encoded in Base62,
> with a KGS leasing disjoint ranges coordinated by ZooKeeper — zero collisions, no DB locks, resilient to KGS/leader
> failure. I default to 302 to keep analytics and control, and pipe clicks fire-and-forget into Kafka → Flink →
> ClickHouse so a viral link is queue depth, not redirect latency. The mapping is sharded by short_code via
> consistent hashing (single-shard redirects), with CDN + local cache + key replication absorbing hot keys. The
> gateway does token-bucket rate limiting and safe-browsing checks; expired links are cleaned lazily on access plus a
> throttled off-peak sweeper / native TTL. Every heavy component is justified by a specific bottleneck — at lower
> scale I'd deliberately remove them."*

← Back to **[URL Shortener overview](02-systems/url-shortener/README.md)** · Base **[deep dives](02-systems/url-shortener/deep-dives.md)** · **[cheatsheet](02-systems/url-shortener/cheatsheet.md)**
