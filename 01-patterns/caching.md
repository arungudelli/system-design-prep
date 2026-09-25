# Caching

> Caching is the single highest-leverage move in most read-heavy designs — and the source of the most subtle
> bugs (stale data, stampedes, the "what if Redis dies" question). Interviewers use it to separate people who
> say *"add a cache"* from people who can say *which* strategy, *why*, and *what breaks*.

**One-line mental model:** a cache trades **freshness** and **memory** for **latency** and **reduced load** on
the source of truth. Every caching decision is a point on that trade.

---

## Problem

The source of truth (a database, a downstream service) is too slow or too expensive to hit for every read.
Reads are repetitive (the same keys, over and over) and can tolerate *some* staleness. We want to serve most
reads from fast memory close to the caller.

## When to use

- **Read-heavy** workloads with **repeated access** to the same keys (high temporal/spatial locality).
- Reads that tolerate **bounded staleness** (a few seconds/minutes is fine).
- Expensive-to-compute or expensive-to-fetch results (aggregations, joins, remote calls).
- You need to **shield the database** from read load (protect the primary, reduce replica count).

## When NOT to use (or be careful)

- **Write-heavy, low-reuse** data — low hit ratio means the cache is pure overhead.
- **Strong-consistency / correctness-critical reads** where stale = wrong (account balance at point of charge,
  inventory at checkout) — cache carefully or not at all on that path.
- **Uniformly random access over a huge key space** — nothing is "hot", so hit ratio stays low.
- When it merely **hides a bad query** you should have indexed/fixed — cache is a multiplier, not a cure.

---

## Where caches live (layers)

You can cache at many layers; they compose:

```text
Client / browser cache        (HTTP cache-control, local storage)
        |
CDN / edge cache              (static assets, cacheable GETs, popular content)
        |
Load balancer / API gateway   (response cache for identical requests)
        |
Local in-process cache        (per-app-server, microseconds, no network hop)   ← great for hot keys
        |
Distributed cache (Redis/Memcached)  (shared across servers, ~sub-ms in-DC)
        |
Database buffer pool           (the DB's own page cache)
```

**Interview tip:** name the layer you mean. "Cache" is ambiguous; "a shared Redis cache-aside layer with a
small local in-process cache in front for hot keys" is a design.

**Local vs distributed:**
- **Local (in-process):** fastest (no network), but each server has its own copy → duplication + per-server
  inconsistency + cold on deploy. Perfect for a **small, very hot set** and for absorbing **hot keys**.
- **Distributed (Redis/Memcached):** shared, consistent-ish across the fleet, larger capacity; costs a network
  hop and is an operational dependency. The default for a shared application cache.

---

## Caching strategies (know all four cold)

### 1. Cache-aside (lazy loading) — the default
The application manages the cache. On read: check cache → on miss, read DB, populate cache, return.

```text
read(key):
  v = cache.get(key)
  if v is None:                      # miss
      v = db.get(key)
      cache.set(key, v, ttl)
  return v

write(key, val):
  db.put(key, val)
  cache.delete(key)                  # invalidate; next read repopulates
```

- ✅ Simple, resilient (cache optional — a miss just hits the DB), only caches what's actually read.
- ❌ First read of a key is a miss (cold-start latency); risk of **stale** data between DB write and cache
  invalidation; needs careful invalidation on writes.
- **Invalidate-on-write (delete), don't update-on-write** — deleting is safer than trying to keep the cache
  value in perfect sync (avoids race conditions where two writers interleave).

### 2. Read-through
The cache library/layer itself loads from the DB on a miss (app only talks to the cache). Same behavior as
cache-aside but the load logic lives in the cache layer, not the app.
- ✅ Cleaner app code; centralized load logic.
- ❌ Needs cache that supports it (or a wrapper); still cold on first read.

### 3. Write-through
Write to cache **and** DB synchronously on every write; reads are always warm.
- ✅ Cache never stale relative to DB; reads always hit.
- ❌ Every write pays cache + DB latency; caches data that may never be read (wasted memory).
- **Great combo:** write-through **on create** + cache-aside for general reads (what the URL shortener does —
  a freshly created code is instantly hot).

### 4. Write-behind (write-back)
Write to cache immediately, **asynchronously** flush to DB later (batched).
- ✅ Fast writes, absorbs write bursts, batches DB writes.
- ❌ **Risk of data loss** if the cache dies before flush; complex; DB temporarily behind cache. Use only when
  some loss is tolerable (metrics counters, view counts) or with a durable write buffer.

| Strategy | Read latency | Write latency | Staleness risk | Data-loss risk | Use when |
|---|---|---|---|---|---|
| Cache-aside | miss = slow | normal | some (until invalidate) | none | general default |
| Read-through | miss = slow | normal | some | none | want clean app code |
| Write-through | always fast | slower | low | none | reads must be warm, writes modest |
| Write-behind | always fast | fast | low | **yes** | write-heavy, loss tolerable |

---

## Eviction policies

Memory is finite; when full, something must go.

- **LRU (Least Recently Used)** — evict the key not touched for longest. Great default; matches temporal
  locality (recently used → likely used again).
- **LFU (Least Frequently Used)** — evict the least-accessed key. Better when popularity is stable over time
  (a genuinely hot key shouldn't be evicted just because it went quiet for a moment).
- **FIFO / TTL-only** — simpler, ignores access pattern.
- **TTL (time-to-live)** — orthogonal to the above: every entry also expires after a set time, bounding
  staleness even if never evicted.

**Interview line:** *"I'd use LRU with a TTL — LRU keeps the working set hot, and the TTL bounds how stale any
entry can get even if it's never overwritten."*

---

## TTL: bound your staleness

- Short TTL → fresher data, lower hit ratio, more DB load.
- Long TTL → higher hit ratio, staler data.
- **Immutable data → very long TTL** (e.g. URL shortener codes never change → cache ~forever).
- **Always add jitter** to TTLs (e.g. `ttl ± random(10%)`) so a batch of keys written together don't all
  **expire at the same instant** → mass miss → stampede.

---

## The three classic cache failure modes (name these — they're common follow-ups)

### 1. Cache stampede (a.k.a. thundering herd / dogpile)
A hot key expires (or the cache restarts cold) and **thousands of concurrent requests all miss and hit the DB
for the same key** at once → DB overload.

**Mitigations:**
- **Single-flight / request coalescing** — only *one* request recomputes/reloads a key; others wait for the result.
- **Jittered TTLs** — don't let many keys expire simultaneously.
- **Early/probabilistic refresh** — refresh a hot key *before* it expires (background or probabilistically as
  TTL nears), so it's never actually cold.
- **Pre-warm** the top-N keys on cache restart/deploy.

### 2. Cache penetration
Requests for keys **that don't exist** always miss the cache and hit the DB every time (accidental or a
malicious scan of random keys).

**Mitigations:**
- **Cache the negative result** (`key → NOT_FOUND`) with a short TTL.
- **Bloom filter** of known keys in front — if the filter says "definitely not present", skip the DB entirely.

### 3. Hot key
One key is *so* popular that the single cache node/shard holding it saturates (CPU/network) — a request-rate
problem, not a data-size problem.

**Mitigations:**
- **Local in-process cache** in front of the distributed cache (each app server serves the hot key from memory).
- **Replicate the hot key** across multiple cache nodes (`key#1..key#N`), read a random replica.
- **CDN/edge** for cacheable hot content.
- **Client-side request coalescing** so a burst becomes one upstream read.

> Related: **hot *partition*** is the same idea one layer down (a DB shard, not a cache node). Fixes rhyme:
> split the key, salt it, or add a caching layer in front. See [sharding](01-patterns/sharding.md) *(coming)*.

---

## "What happens if the cache (Redis) goes down?" (you WILL be asked)

The staff answer has three parts:

1. **Degrade, don't fail.** Reads fall through to the DB / read replicas — slower but **correct**. So the DB
   tier must be sized to survive at least a burst of full read load. Use a **short cache connect-timeout** so a
   dead cache doesn't *add* latency (don't block waiting on it).
2. **Protect the recovery.** A cold cache coming back = a stampede risk. Use single-flight + jittered TTLs +
   pre-warming so the DB isn't hammered as the cache refills.
3. **Don't let it cascade.** If the DB *can't* absorb full read load, add **load shedding / rate limiting** so a
   cache outage degrades gracefully instead of taking down the DB (which would be a bigger outage). A **circuit
   breaker** around the cache client avoids piling up on a dead dependency.

**Anti-pattern to call out:** a cache you *can't* survive losing isn't a cache — it's an undocumented primary
datastore with no durability. If losing the cache takes the system down, that's a design smell.

---

## Consistency: cache is a copy, and copies go stale

- **Invalidate on write** (delete the key) rather than update — simpler and avoids interleaving races.
- **Write-through** keeps cache and DB in lockstep at write time.
- **TTL** is your backstop: even if an invalidation is missed, staleness is bounded.
- **Read-your-writes:** after a user's own write, either write-through so they see it, or route their next read
  to the source of truth for a short window.
- **The dual-write trap:** writing to DB and cache as two steps can leave them inconsistent if one fails.
  Prefer *DB write → cache delete*; on failure, the TTL eventually heals it. For strict cases, treat the DB as
  truth and the cache as best-effort.

---

## Architecture (typical shape)

```text
        Client
          |
   [ App servers ]
     |         \
 local cache    \  (hot keys, microseconds)
     |           \
 distributed cache (Redis) ──miss──► Database (primary + replicas)
   cache-aside, LRU+TTL+jitter        source of truth
```

- Local cache absorbs hot keys and shaves the network hop.
- Redis cache-aside absorbs the bulk of reads (aim 90%+ hit).
- DB serves misses; must survive a cache outage.

---

## Trade-offs (the ones to argue)

- **Cache vs read replicas:** cache is cheaper per read and faster, but adds a consistency/invalidation problem;
  replicas give you consistent-ish reads at more cost and replication lag. Usually **cache first, replicas next**.
- **Local vs distributed:** local is faster but duplicated/inconsistent; distributed is shared but a network hop
  + an operational dependency. Often **both** (local in front of distributed).
- **Freshness vs hit ratio:** short vs long TTL. Tie the choice to how stale the data may safely be.
- **Write-through vs cache-aside:** warm reads + slower writes vs lazy + possible cold misses. Combine them.
- **More memory vs higher miss rate:** bigger cache = higher hit ratio = diminishing returns; size to the hot set.

---

## Interview wording

> "Reads are ~100× writes here, so I'll put a cache-aside Redis layer in front keyed by X, targeting a 90%+ hit ratio."
> "The data is immutable, so I can use a long TTL and cache aggressively — invalidation isn't even a concern."
> "I'll invalidate on write by deleting the key, not updating it, to avoid write-write races; the TTL is my backstop."
> "For the viral hot key, I'd add a small local in-process cache on each server so we don't hammer one Redis node."
> "If Redis goes down we fall through to the DB and degrade gracefully — the DB is sized for that, and I guard the
> cold-start with single-flight and jittered TTLs to avoid a stampede."
> "I'd cache negative lookups with a short TTL, and put a Bloom filter in front, to stop penetration from
> unknown keys hammering the database."

---

## Where it appears (across systems)

- **URL shortener** — cache-aside on `code → URL`, write-through on create, hot-key handling for viral links.
  See [URL Shortener deep dives](02-systems/url-shortener/deep-dives.md#2-caching-strategy).
- **News feed** — cache precomputed timelines; fan-out-on-write populates the cache.
- **Rate limiter** — Redis holds counters/token buckets (a cache *is* the datastore here).
- **Product/catalog reads, sessions, config, autocomplete** — classic cache-aside targets.
- **CDN** for static/media — caching at the edge (see [object storage & CDN](01-patterns/object-storage.md) *(coming)*).

---

## Quick checklist

- [ ] Which **layer** (browser/CDN/gateway/local/distributed)?
- [ ] Which **strategy** (cache-aside default; write-through on create; write-behind only if loss OK)?
- [ ] **TTL** + **jitter** + **eviction** (LRU+TTL default)?
- [ ] **Invalidation** on write (delete, not update)?
- [ ] **Stampede / penetration / hot-key** guards?
- [ ] **What happens if the cache dies** — does the system degrade or fall over?
- [ ] Is any **correctness-critical** read wrongly served from cache?

← Back to **[patterns](/)** · Related: [sharding](01-patterns/sharding.md) *(coming)* · [rate limiting](01-patterns/rate-limiting.md) *(coming)*
