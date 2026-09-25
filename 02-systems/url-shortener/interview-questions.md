# URL Shortener — Interview Questions

> **How to use this:** read each question and *answer it out loud (or write it) first*. Only then expand the
> model answer and critique yourself against the [critique lens](00-framework/staff-level-thinking.md#how-your-answer-gets-graded-the-critique-lens).
> The goal is to *derive*, not recall.

---

## Core / warm-up

**Q1. Why is this system read-heavy, and how does that shape the whole design?**
<details><summary>Model answer</summary>

People click links far more than they create them (~100:1). So the **read/redirect path** is the thing to
optimize: cache-first (Redis, cache-aside, long TTLs because codes are immutable), read replicas behind the
cache, a thin synchronous redirect handler, analytics pushed off the critical path. Writes (~120/s) are trivial —
no sharding, single primary is fine. Naming reads as dominant *is* the design.
</details>

**Q2. How long should the short code be, and why?**
<details><summary>Model answer</summary>

**7 base62 chars.** 62⁷ ≈ 3.5 trillion. We need ~6B codes over 5 years, so 7 chars gives ~570× headroom.
6 chars (56B) works but leaves little margin; 8 is future-proof but longer. Show the math — that's the point.
</details>

**Q3. How do you generate unique codes without collisions?**
<details><summary>Model answer</summary>

**Counter + base62**, distributed via a **Key Generation Service** that leases disjoint ranges to each app
server (A: 1–1M, B: 1M–2M…). Servers mint codes from their in-memory range → collision-free, no hot-path
coordination, no DB read on create. Avoid hash-truncation (collisions → read-and-retry per create). If codes
must be unguessable, run the counter through a reversible permutation. See [Deep Dives](02-systems/url-shortener/deep-dives.md#1-unique-short-code-generation-the-core-sub-problem).
</details>

**Q4. SQL or NoSQL — and defend it.**
<details><summary>Model answer</summary>

The access pattern is a **keyed point-lookup, no joins** → both fit. At ~120 writes/s and ~600 GB/yr, a
**single SQL primary + read replicas + cache** is simplest and gives free constraints (custom-alias uniqueness).
A KV store (DynamoDB) is the natural fit if we want hands-off horizontal/global scale. Decide by **access
pattern + team stack**, not popularity. Don't say "NoSQL is faster."
</details>

---

## Redirect & caching

**Q5. 301 or 302? What are you trading?**
<details><summary>Model answer</summary>

**302 by default.** 301 lets browsers/CDNs cache the mapping → lower origin load/latency, but you **lose
per-click analytics** and the ability to change/expire the link. 302 sends every click through you → analytics
+ control at the cost of more traffic. It's control/analytics vs offload/latency — choose 302 unless offload
matters more than tracking for specific links.
</details>

**Q6. What happens if Redis (the cache) goes down?**
<details><summary>Model answer</summary>

Reads **fall through to DB + read replicas** — slower but correct; the system **degrades, not fails**. So the
DB tier must be sized to survive a cache outage. On cache recovery (cold), guard against **stampede** with
single-flight per key + jittered TTLs + pre-warming the top-N. And cache **negative** results (+ optional Bloom
filter) so unknown-code floods (**penetration**) don't hammer the DB.
</details>

**Q7. A single link goes viral — one code is 40% of your traffic. What breaks and what do you do?**
<details><summary>Model answer</summary>

That's a **hot key**: one cache node saturates on request *rate* (the value is tiny). Fix with a **local
in-process cache** on each app server for top-N codes (coalesces requests), **replicate the hot key** across
cache nodes, and/or let a **CDN** absorb it at the edge. Detect via per-key hit rate + a single node running hot.
</details>

**Q8. Why is the analytics write not on the critical path, and how do you wire it?**
<details><summary>Model answer</summary>

The user is waiting on the redirect; making them wait on an analytics write would blow the latency budget for no
user benefit. So on redirect we **emit a click event to a queue** (Kafka/SQS) and return the 302 immediately;
consumers aggregate asynchronously (eventually consistent counts, HyperLogLog for uniques). This is also *why*
we prefer 302 — a cached 301 means most clicks never reach us to count.
</details>

---

## Consistency & correctness

**Q9. Is eventual consistency acceptable here? Where is it NOT?**
<details><summary>Model answer</summary>

Yes for reads — codes are **immutable**, so a ~second of replication/cache propagation after create is fine
(and if "read-your-writes" is needed right after create, write-through the cache so the creator sees it
instantly). Where it's *not* acceptable: **uniqueness of codes/custom-aliases** — that must be strongly enforced
(unique constraint / conditional write) or you'd hand two links the same code.
</details>

**Q10. `POST /urls` times out and the client retries. How do you avoid creating two codes?**
<details><summary>Model answer</summary>

**Idempotency-Key**: client sends a UUID; server stores `key → shortCode` and returns the same code on replay.
(Alternatively, if the product wants one-code-per-URL, a unique index on `long_url` makes it naturally
idempotent.) Most shorteners want distinct codes per request, so prefer the idempotency key for retry-safety.
</details>

**Q11. Can you ever reuse a code (e.g. from a crashed server's range, or an expired link)?**
<details><summary>Model answer</summary>

**No — avoid reuse.** Reusing a code risks pointing an old, still-shared link at a brand-new target (a
correctness/security problem). A crashed server's unused range is simply **skipped** — codes are cheap
(trillions available). Expired codes can be tombstoned but shouldn't be recycled into new mappings.
</details>

---

## Scale & evolution

**Q12. Walk the create flow and the redirect flow end to end.**
<details><summary>Model answer</summary>

**Create:** validate (+ optional safe-browsing) → get code from KGS range → INSERT → write-through cache →
201. **Redirect:** cache lookup (≈90% hit) → on miss read DB by PK + populate cache → 302 with `Location` →
async click event. Redirect is synchronous/latency-critical; analytics is async. See [Architecture](02-systems/url-shortener/architecture.md#10-requestdata-flows).
</details>

**Q13. What's the first bottleneck, and how do you scale it — in order?**
<details><summary>Model answer</summary>

**Read throughput.** Order: (1) cache-aside Redis (~90% hit, biggest win), (2) read replicas for misses,
(3) CDN/edge for popular codes at global scale. Storage and bandwidth are **non-issues** (say so) — don't shard
for size. The only real scaling axis is reads.
</details>

**Q14. What changes at 10×? At 100×?**
<details><summary>Model answer</summary>

**10×** (~120k reads/s): more cache nodes (watch for hot keys), more read replicas, likely add a CDN/edge layer.
Writes (~1.2k/s) still trivial. **100×** (~1.2M reads/s): CDN does the heavy lifting for popular codes;
multi-region read replicas near users; KGS partitions the counter space per region so global creation stays
collision-free. The mapping being immutable makes global read replication genuinely easy — that's the unlock.
</details>

**Q15. When, if ever, would you shard the database — and on what key?**
<details><summary>Model answer</summary>

Probably **never for this system** at realistic scale — storage (~600 GB/yr) and writes (~120/s) don't demand
it, and cache + replicas handle reads. *If* forced (e.g. writes grew 1000×), shard by **`short_code`** (hash-
based) since the dominant query is a point lookup by it — that keeps every read on a single shard. Range-based
sharding on a monotonic counter would create a **hot shard** at the tail, so prefer hash/consistent-hashing.
Say: "I'd exhaust caching and replicas first."
</details>

---

## Failure & operations

**Q16. The DB primary fails over. What's the user impact on reads vs creates?**
<details><summary>Model answer</summary>

**Reads: unaffected** — served from cache + read replicas. **Creates: pause** (seconds) until a replica is
promoted, returning `503 + Retry-After`. That's acceptable because our create availability target is only 99.9%
while reads are 99.99%. Ensure KGS range state is durable so code generation resumes cleanly.
</details>

**Q17. A bad deploy ships a broken redirect handler. How do you limit the blast radius?**
<details><summary>Model answer</summary>

Redirect is the crown jewel — a break there kills *every* link. **Canary** a small % of traffic, watch p99 +
redirect success + error rate, **auto-rollback** on regression. Keep the handler tiny and boring to minimize
churn. Make schema changes backward-compatible (expand/contract) so old and new versions coexist during rollout.
</details>

**Q18. The create path calls an external safe-browsing API that slows from 200 ms to 10 s. What happens and what do you do?**
<details><summary>Model answer</summary>

Synchronous calls pile up → thread/connection exhaustion on the create path. Wrap it in **timeout + circuit
breaker + bulkhead**. On breaker-open, choose fail-closed (reject creates) or fail-open (accept + async re-scan)
per risk appetite. Crucially, keep the **redirect path free of any external calls** so this can never affect reads.
</details>

**Q19. How do you prevent someone enumerating all links by walking sequential codes?**
<details><summary>Model answer</summary>

Sequential counter codes are enumerable. If privacy is required, **don't expose the raw integer** — run it
through a reversible permutation (Feistel / multiply-XOR mod 62⁷) so codes stay unique and collision-free but
non-sequential, or interleave random base62 chars. Also rate-limit and monitor for scanning patterns. Only do
this if unguessability is a stated requirement.
</details>

---

## Staff-level curveballs

**Q20. The event/analytics store is down. Should redirects fail? Justify.**
<details><summary>Model answer</summary>

**No.** Analytics is off the critical path; redirects must never depend on it. Buffer click events locally with
a bounded queue and drop-oldest (analytics tolerates loss), and let backpressure die at the buffer — it must
never reach the redirect. Availability of the core function outranks completeness of analytics.
</details>

**Q21. Product wants "same long URL always returns the same short code." What changes, and what new problems appear?**
<details><summary>Model answer</summary>

Add a **unique index on `long_url`** (or look up before insert) so creation dedupes. New problems: (1) two users
now **share** a link + its analytics and neither can independently expire it; (2) the lookup adds a read on
create; (3) normalization questions (is `example.com/a` == `example.com/a/` == with tracking params?). Flag these
trade-offs rather than silently implementing — it changes ownership semantics.
</details>

**Q22. How would you run this active-active across two regions? What's the one hard part?**
<details><summary>Model answer</summary>

The read path is easy: the mapping is **immutable/read-only**, so replicate it to both regions and serve reads
locally (route by geo/anycast). The **one hard part is key generation** — two regions must never mint the same
code. Solve by **partitioning the counter space per region** (region A uses even ranges / a region prefix,
region B odd, etc.) so codes are globally unique with zero cross-region coordination on the hot path.
</details>

**Q23. Estimate the QPS and storage from scratch, out loud.**
<details><summary>Model answer</summary>

100M writes/month ÷ 2.5M s ≈ 40 w/s (×3 peak ≈ 120). Reads at 100:1 ≈ 4k/s (≈12k peak). Storage: 500 B ×
100M/mo × 12 ≈ 600 GB/yr. Cache hot set ~10 GB → 90%+ hit. Conclusion stated: reads dominate → cache + replicas;
writes/storage trivial → no sharding. See [Capacity](02-systems/url-shortener/capacity.md).
</details>

**Q24. What would you deliberately keep simple, forever — and why?**
<details><summary>Model answer</summary>

The **redirect handler** and the **data model** (a single keyed table). They're the hot path and the source of
truth; complexity there is pure risk with no upside. Keep cleverness in the *edges* (cache, KGS, analytics
pipeline), not in the thing that must be correct and fast a million times a second.
</details>

**Q25. When would you redesign this system?**
<details><summary>Model answer</summary>

When a **core assumption breaks**: it becomes write-heavy (e.g. machine-generated links at massive scale →
revisit write path/sharding), or it must be strongly consistent + low-latency **globally** (revisit key-gen
partitioning and replication), or analytics becomes a first-class product (a real streaming pipeline, not a
side queue). Absent those, this design holds for years — which is the point of starting simple.
</details>

---

## Self-scoring

After a mock, grade yourself honestly on the [15-point critique lens](00-framework/staff-level-thinking.md#how-your-answer-gets-graded-the-critique-lens):
did you clarify first, quantify the scale, justify every component, name the dominant bottleneck, handle failure,
and argue trade-offs both ways? Write down the **one weakest area** and drill it before the next system.

← Back to **[README](02-systems/url-shortener/README.md)** · Next: **[Cheatsheet](02-systems/url-shortener/cheatsheet.md)**
