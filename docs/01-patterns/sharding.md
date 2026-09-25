# Partitioning & Sharding

> Sharding is the **last** database-scaling tool you reach for — and the one candidates reach for *first*,
> which is exactly the mistake interviewers watch for. It's powerful and expensive: it buys near-unlimited
> horizontal scale at the cost of cross-shard queries, transactions, hot shards, and painful rebalancing.
> The staff move is to **exhaust the cheaper options first** and shard only when a real bottleneck forces it.

**One-line mental model:** *partitioning* splits one big dataset into smaller pieces; *sharding* is
partitioning across **separate machines** so each holds only a slice of the data and the traffic.

---

## Vocabulary (say these precisely)

- **Partitioning** — dividing a dataset into parts (partitions). Can be within one machine.
- **Sharding** — partitioning **horizontally across multiple nodes**; each node ("shard") owns a subset of rows.
  Usually "sharding" implies horizontal partitioning across machines.
- **Horizontal partitioning** — split by **rows** (users 1–1M on shard A, 1M–2M on shard B). This is sharding.
- **Vertical partitioning** — split by **columns/tables** (put the big `blob` column or a hot table on its own
  store). Different tool, different problem.
- **Shard key / partition key** — the column whose value decides which shard a row lives on. **The single most
  important choice** — get it wrong and everything downstream hurts.

---

## The database-scaling ladder (cheapest first — ALWAYS in this order)

Shard only after you've climbed past the earlier rungs. For each rung: *what it solves*, *when you need it*,
*what new problem it introduces*.

### 0. Fix the query / add indexes
- **Solves:** slow reads from full scans.
- **Need it when:** query latency high but data volume modest.
- **Introduces:** indexes slow writes slightly and use space. Often this alone buys years.

### 1. Cache
- **Solves:** repeated reads hammering the DB.
- **Need it when:** read-heavy with locality (see [caching](caching.md)).
- **Introduces:** staleness/invalidation, "what if the cache dies", stampedes.

### 2. Read replicas (replication)
- **Solves:** read *throughput* — spread reads across copies; primary handles writes.
- **Need it when:** reads exceed what one node serves *after* caching.
- **Introduces:** **replication lag** → stale reads on replicas (read-your-writes issues); doesn't help **writes**.

### 3. Vertical partitioning / bigger box
- **Solves:** one hot table or a huge column; or just "not enough CPU/RAM".
- **Need it when:** a specific table/column dominates, or vertical scaling is still cheaper than sharding.
- **Introduces:** eventually you hit the ceiling of a single machine.

### 4. Partitioning (single DB, multiple partitions)
- **Solves:** table too big to manage; enables partition pruning, easier archival (drop old partitions).
- **Need it when:** one table is huge but still fits one machine.
- **Introduces:** partition key choices; still one machine's write ceiling.

### 5. Sharding (partition across machines) ← the last resort
- **Solves:** **write throughput** and **dataset size** beyond one machine.
- **Need it when:** writes (or storage) exceed what a single primary can handle, *after* all the above.
- **Introduces:** cross-shard queries/transactions, hot shards, resharding/rebalancing, operational complexity.

> **Interview line:** *"Before sharding I'd first exhaust indexing, caching, and read replicas. Sharding solves
> write and storage scale that those can't — but it introduces cross-shard queries, hot shards, and rebalancing,
> so I only take it on when writes or data volume genuinely exceed a single primary."*

---

## Problem (why shard at all)

A single primary DB has a hard ceiling on **write throughput**, **storage**, **connections**, and **memory**.
Caching and replicas scale *reads*, not writes. When your write rate or dataset outgrows the biggest single box
you're willing to run, you must spread **writes and data** across multiple independent nodes → sharding.

## When to use
- **Write throughput** exceeds a single primary (after caching reads away).
- **Dataset size** exceeds one machine's storage/memory.
- You need **blast-radius isolation** (a shard outage affects only its slice) or **tenant isolation**.

## When NOT to use
- Reads are the problem → cache + replicas first.
- Data fits comfortably on one machine and writes are modest (e.g. the URL shortener: ~120 w/s, ~600 GB/yr →
  **never shard**). Sharding here is pure cost, no benefit.
- You haven't identified a shard key that matches your access patterns → sharding will create cross-shard pain.

---

## Sharding strategies (know all four, and their failure modes)

### 1. Range-based sharding
Assign contiguous key ranges to shards (A: users 1–1M, B: 1M–2M, …), or by date/alphabet.

- ✅ **Range queries are efficient** (`WHERE id BETWEEN …` hits few shards); simple to reason about.
- ❌ **Hot shards**: sequential/monotonic keys (auto-increment IDs, timestamps) send *all new writes to the last
  shard* → one node is hot while others idle. Uneven data distribution.
- **Use when:** you need range scans (time-series where you query recent windows) *and* you can avoid a
  monotonic hot tail (e.g. shard by a composite key).

### 2. Hash-based sharding
`shard = hash(key) % N`. Spreads keys uniformly.

- ✅ **Even distribution**, no hot shard from monotonic keys; simple point lookups.
- ❌ **Range queries are gone** (adjacent keys scatter across shards). And the classic killer: **`% N` means
  changing N (adding/removing a shard) remaps almost every key** → massive data movement. → fixed by consistent hashing.
- **Use when:** access is point-lookup by key and you don't need range scans (e.g. shard the URL shortener by
  `hash(short_code)` if ever forced to).

### 3. Consistent hashing (the fix for "adding a node remaps everything")
Map both keys and nodes onto a ring (hash space). A key belongs to the next node clockwise. Adding/removing a
node only moves the keys **between that node and its neighbor** — roughly **1/N of keys**, not all of them.

```text
        key3
   nodeC •───────• nodeA
        /  ring    \
   key2 •           • key1
        \          /
         •────────•
           nodeB
Add nodeD between C and A → only keys in that arc move to D. Everyone else stays put.
```

- ✅ **Minimal remapping** on scale changes → smooth elasticity. The default for distributed caches (Redis
  Cluster, Memcached clients), DynamoDB/Cassandra partitioning.
- ⚠️ Naive placement can be **uneven** → use **virtual nodes** (each physical node owns many small arcs) to
  smooth distribution and make rebalancing granular.
- **Where it appears:** distributed caches, Cassandra, DynamoDB, load balancing across a changing node pool.

> **This is the #1 consistent-hashing interview point:** *"With `hash % N`, adding the (N+1)th node remaps
> almost every key — catastrophic cache churn / data movement. Consistent hashing moves only ~1/N of keys, and
> virtual nodes keep the distribution even."*

### 4. Directory-based (lookup table) sharding
A lookup service maps `key → shard` explicitly (a directory/metadata service).

- ✅ **Maximum flexibility** — move any key to any shard, rebalance precisely, mix strategies. Great for
  multi-tenant ("tenant X lives on shard 7").
- ❌ The directory is an **extra hop and a potential SPOF/bottleneck** → must be cached and made HA.
- **Use when:** you need fine-grained control or tenant placement, and can operate a reliable directory.

| Strategy | Even distribution | Range queries | Rebalance cost | Notes |
|---|---|---|---|---|
| Range | ❌ (hot tail risk) | ✅ efficient | moderate | good for time-series with care |
| Hash `% N` | ✅ | ❌ | **terrible** (remaps all) | avoid at scale |
| Consistent hashing | ✅ (with vnodes) | ❌ | **low** (~1/N) | default for elastic clusters |
| Directory | ✅ (you control) | depends | **flexible** | extra hop; SPOF risk; great for tenants |

---

## Choosing the shard key (the decision everything hinges on)

A good shard key is:

1. **High cardinality** — many distinct values so load spreads (don't shard by `country` if 90% are one country).
2. **Even access distribution** — no single value that's disproportionately hot.
3. **Aligned with the dominant query** — so the common query hits **one shard** (avoid scatter-gather).
4. **Stable** — a value that doesn't change (moving a row between shards is expensive).

Examples:
- **Shard by `user_id`** — most queries are "this user's data" → single-shard reads. Common default.
- **Multi-tenant:** shard by `tenant_id` → tenant isolation, but watch the **whale tenant** (one tenant = 50% of
  load = hot shard) → may need to sub-shard big tenants or place them alone (directory).
- **Time-series:** shard by `(metric_id, time_bucket)` not raw time, to avoid a monotonic hot tail.

> **Trap:** picking a shard key that's great for one query but forces **scatter-gather** (query every shard,
> merge results) for another common query. If two dominant queries want different shard keys, you may need a
> secondary index / a second copy sharded differently — a real cost to name.

---

## The hard parts (this is where the interview goes deep)

### Hot shards / hot partitions
One shard gets disproportionate load (bad key choice, a whale tenant, a viral entity, or a monotonic range tail).
- **Detect:** per-shard QPS/CPU/storage skew; one node's latency diverging.
- **Fix:** better shard key; **salt** the hot key (`key#bucket`) to spread it; **split** the hot shard; put a
  **cache** in front of the hot entity (see [caching hot keys](caching.md#3-hot-key)); isolate the
  whale via directory placement.

### Resharding / rebalancing (the operational nightmare)
Adding shards (or fixing skew) means **moving data while serving traffic**.
- **`hash % N` → avoid**, because it moves almost everything. Use **consistent hashing (with vnodes)** so only
  ~1/N moves, or a **directory** so you move precisely.
- **Live migration pattern:** dual-write to old+new location → backfill/copy historical data → verify →
  cut reads over → stop writing old → clean up. Do it incrementally, key-range by key-range.
- **Pre-sharding:** create many more logical shards than physical nodes up front (e.g. 1024 virtual shards on 4
  nodes); "adding a node" just **moves whole virtual shards**, no rehashing of individual keys. Very common in practice.

### Cross-shard queries (scatter-gather)
A query that spans shards must hit many shards and merge results → slow, hard to paginate/sort, amplifies load.
- **Minimize** by choosing a shard key aligned to the dominant query.
- **Denormalize / secondary index:** keep a second copy sharded by the other access pattern.
- **Aggregate offline:** push cross-cutting analytics to a warehouse / search index rather than the OLTP shards.

### Cross-shard transactions
ACID across shards is expensive and usually avoided.
- **Prefer:** keep a transaction **within a single shard** (co-locate related data under the same shard key).
- **When you can't:** use the **[saga pattern](idempotency.md)** *(coming)* (a sequence of local
  transactions + compensating actions) or a **transactional outbox** + events for eventual consistency. Reserve
  2-phase-commit for rare cases — it's slow and blocks on coordinator failure.
- **Say:** *"I'd design the shard key so the transaction stays on one shard; if it truly can't, I'd use a saga
  with idempotent steps rather than a distributed transaction."*

### Global secondary indexes & uniqueness
A `UNIQUE(email)` constraint is trivial on one DB, hard across shards (the email may hash to a different shard
than the user row). Options: a separate index table sharded by the indexed column, or a dedicated uniqueness
service. Name this cost — interviewers probe it.

### Rebalancing + replication interplay
Each shard is itself usually **replicated** (primary + replicas) for HA. So "sharded" almost always means
"sharded **and** replicated" → N shards × R replicas. Don't forget failover *per shard*.

---

## Architecture (typical sharded setup)

```text
              Client
                |
         [ App / query router ]  ── knows shard map (or asks directory)
           /      |       \
     Shard A   Shard B   Shard C      each = primary + replicas
     (keys …)  (keys …)  (keys …)
     + own replication + failover
        \        |        /
       (optional) directory / metadata service: key -> shard
```

- **Routing** lives in the app, a proxy (e.g. Vitess, a smart client), or via a **directory**.
- Each shard is independently replicated and fails over on its own.
- Cross-shard reads = scatter-gather at the router; avoid on the hot path.

---

## Trade-offs (argue both sides)

- **Shard vs bigger box:** sharding = near-unlimited scale + blast-radius isolation, but huge operational
  complexity. A bigger single box is simpler and often cheaper until you truly outgrow it. *Vertical first.*
- **Hash vs range:** even load vs range-query support. Pick by whether you need range scans.
- **Consistent hashing vs directory:** automatic elastic rebalance vs precise manual control (+ a SPOF to run).
- **Fewer big shards vs many small (pre-sharded) shards:** fewer = less overhead; many = easier rebalancing and
  finer hot-shard splitting. Pre-sharding is usually worth it.
- **Co-locate for single-shard transactions vs spread for even load:** the shard-key tension in one line.

---

## Interview wording

> "The data fits one machine and writes are modest, so I would **not** shard — that's premature complexity."
> "Reads are the bottleneck, so I'll cache and add read replicas before ever considering sharding."
> "If writes outgrow a single primary, I'd shard by `user_id` so a user's data lives on one shard and the common
> query stays single-shard."
> "I'd use consistent hashing with virtual nodes so adding a node moves only ~1/N of the keys, not all of them."
> "The risk with `tenant_id` sharding is a whale tenant becoming a hot shard — I'd isolate big tenants via a
> directory or sub-shard them."
> "I'd keep transactions within a shard by co-locating related data; if a cross-shard operation is unavoidable,
> I'd use a saga with idempotent, compensatable steps rather than a distributed transaction."
> "For rebalancing I'd pre-shard into many virtual shards so scaling out just relocates whole virtual shards."

---

## Where it appears (across systems)

- **URL shortener** — the *counter-example*: numbers prove you **don't** shard (cache + replicas suffice).
  See [URL Shortener Q15](../02-systems/url-shortener/interview-questions.md).
- **Chat / messaging** — shard by `conversation_id` so a conversation's messages co-locate.
- **News feed** — shard user/timeline data by `user_id`; celebrities become hot shards → hybrid handling.
- **Metrics/time-series** — shard by `(series, time_bucket)` to avoid a monotonic hot tail.
- **Distributed cache / Cassandra / DynamoDB** — consistent hashing + virtual nodes under the hood.

---

## Quick checklist

- [ ] Have I **exhausted indexing, caching, replicas** first? (If not, don't shard.)
- [ ] Is the bottleneck **writes or storage** (shard) or **reads** (cache/replicas)?
- [ ] **Shard key:** high-cardinality, evenly accessed, aligned to the dominant query, stable?
- [ ] **Strategy:** range (range scans) / hash (even, point) / consistent hashing (elastic) / directory (control)?
- [ ] **Hot shard** risk (monotonic keys, whale tenants, viral entities) — mitigation?
- [ ] **Cross-shard** queries/transactions — minimized or handled (saga/denormalize)?
- [ ] **Rebalancing** plan (pre-sharding / consistent hashing / dual-write migration)?
- [ ] Each shard **replicated + failover**?

← Back to **[patterns](../index.md)** · Related: [caching](caching.md) · [idempotency](idempotency.md) *(coming)*
