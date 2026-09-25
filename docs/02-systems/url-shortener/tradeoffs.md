# URL Shortener — Trade-offs

> The rule: **never declare one option "better."** State what each buys you, what it costs, and which one wins
> *given the dominant requirement* (here: low redirect latency + high read availability).

---

## 1. Code generation: Counter+base62 (KGS) vs Hash-truncation vs Random

| | Counter + base62 (KGS) | Hash-truncation | Random + check |
|---|---|---|---|
| Collisions | **None** (disjoint ranges) | Inevitable → read+retry | Rare, rise as space fills → check needed |
| Hot-path cost | In-memory increment | DB read to check collision | DB read to check |
| Guessable? | Yes (mitigate w/ permutation) | No | No |
| Coordination | KGS leases ranges (cheap) | None (stateless) | None |
| Operational | One small HA service | Simplest | Simple |

**Choose:** Counter+base62 via KGS as default (collision-free, no hot-path DB read). Add a reversible
permutation if unguessability is required. Use random+check only when unguessability matters more than the
extra write-time lookup. → detail in [Deep Dives](deep-dives.md).

---

## 2. SQL vs NoSQL

| | SQL (Postgres/MySQL) | NoSQL KV (DynamoDB/Cassandra) |
|---|---|---|
| Fit for keyed lookup | Great (PK lookup) | Great (partition key = O(1)) |
| Scale writes | Vertical + replicas (enough here) | Horizontal by default |
| Constraints (alias uniqueness) | **Free** (unique index, transactions) | App-enforced / conditional writes |
| Operational at this scale | Very simple | Simple, auto-scaling |
| When it wins | Team runs SQL; want constraints; volume modest (our case) | Extreme scale, global tables, hands-off scaling |

**Choose:** Either is defensible. At ~120 writes/s and ~600 GB/yr, **a single SQL primary + read replicas +
cache is simplest and gives free constraints** — a great default. Reach for a KV store if you expect
DynamoDB-style hands-off global scale or the org already standardizes on it. **Justify by access pattern + team,
not popularity.** → SQL vs NoSQL *(coming)*.

---

## 3. 301 vs 302 redirect

- **301 (permanent):** browsers/CDNs cache it → **lower origin load & latency**, but you **lose per-click
  analytics** and the ability to change/expire the link.
- **302 (temporary):** every click reaches you → **full analytics + control**, at the cost of **more traffic**.

**Choose:** **302 by default** (analytics + control are usually product requirements). Use 301 only for links
where offload matters more than tracking. → table in [Deep Dives](deep-dives.md#3-redirect-301-vs-302-a-genuine-trade-off--expect-a-follow-up).

---

## 4. Cache-aside vs Write-through

- **Cache-aside (reads):** app populates cache on miss. Simple, resilient (cache optional), but first read of a
  cold key is a miss.
- **Write-through (on create):** new mapping written to cache immediately → the "create then share" path is hot
  from click #1.

**Choose:** **Both** — cache-aside for the general read path, write-through on create. Cheap and complementary.

---

## 5. Dedupe by long URL: yes vs no

- **Dedupe (one code per URL):** saves keys, but two users shortening the same URL share analytics and one can't
  expire without affecting the other.
- **No dedupe (code per request):** clean ownership + separate analytics; uses more keys (we have trillions).

**Choose:** **No dedupe** by default (independent links). Offer dedupe only if the product explicitly wants
"same URL → same code." Use an **Idempotency-Key** for retry-safety instead of URL-dedup.

---

## 6. Single-region vs Multi-region

- **Single region + multi-AZ:** simple, resilient to AZ failure, adequate for most launches.
- **Multi-region:** lower global read latency + regional resilience. **Easy for this system** because the
  mapping is immutable/read-only → replicate everywhere. The only real coordination is **key generation**
  (partition the counter space per region so codes never collide).

**Choose:** Start **single-region multi-AZ**. Go multi-region when global latency or regional-outage resilience
is a stated requirement — and note it's unusually cheap to do here. → active-active vs active-passive *(coming)*.

---

## 7. Sync vs async work

- **Redirect + create → synchronous** (user is waiting; both are fast).
- **Analytics, safe-browsing re-scan, cleanup → asynchronous** (off the critical path).

**Choose:** keep the user-facing path synchronous and thin; push everything non-essential to a queue.

---

## The one-paragraph summary (say this)

> *"Reads dominate 100:1 and must be sub-50 ms, so the design is cache-first: Redis cache-aside with long TTLs
> (codes are immutable), read replicas behind it, and a single primary for the trivial write load. I generate
> codes with a counter encoded in base62, handed out in ranges by a small Key Generation Service, so creation
> is collision-free and contention-free. I default to 302 redirects to keep analytics and control. SQL or a KV
> store both fit the keyed-lookup access pattern; I'd pick based on the team's stack. Storage and bandwidth are
> non-issues at this scale, so I don't shard — the only real scaling axis is read throughput, which the cache
> and replicas handle, with a CDN/edge layer if we go global."*

→ Next: **[Failure Scenarios](failure-scenarios.md)**
