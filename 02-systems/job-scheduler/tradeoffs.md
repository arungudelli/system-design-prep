# Distributed Job Scheduler — Trade-offs

> Dominant requirements: **reliable, near-on-time execution** with **exactly-once effect** and a **durable
> schedule**. Judge every trade against those.

---

## 1. Poll (pull) vs push scheduling

- **Poll:** scheduler queries the due-index every tick. Simple, robust, easy to reason about; timing precision =
  tick interval; wasted queries when nothing's due.
- **Push / timer-driven:** schedule an OS/timer-wheel event per job to fire at exactly its time. Precise, no polling
  waste; but harder to make durable + distributed (timers live in memory; a crash loses them) and awkward at millions
  of jobs.

**Choose:** **poll a durable due-index** for a distributed system (durability + simplicity win). Use in-memory
timer wheels only for a single-node, sub-second, non-durable scheduler. Hybrid: poll to load the next window into an
in-memory timer wheel for precision, backed by the durable store.

---

## 2. Separate scheduler + queue + workers vs monolithic scheduler-executor

- **Separated (recommended):** scheduler decides *when*, queue buffers, workers *execute*. Independent scaling +
  failure isolation + bursts buffered; more moving parts.
- **Monolithic:** the scheduler also runs the jobs. Simpler to deploy; but a slow/heavy job blocks scheduling, and
  you can't scale execution independently.

**Choose:** **separate** for anything at scale — the decoupling is the core architectural win. Monolith only for tiny
scale. See [architecture](02-systems/job-scheduler/architecture.md#the-core-insight-separate-scheduling-from-execution).

---

## 3. Exactly-once effect vs true exactly-once delivery

- **Chase exactly-once delivery:** a trap — impossible across process + ack boundaries.
- **Exactly-once effect** = at-least-once + idempotency + fencing.

**Choose:** **exactly-once effect.** Say it out loud; it's the mature answer. → [deep dive](02-systems/job-scheduler/deep-dives.md#2-exactly-once-execution-the-honest-version).

---

## 4. Lease + fencing vs distributed lock service

- **Lease in the DB (status/lease_expires_at/fence_token):** no extra infra; the schedule store *is* the lock.
  Simple, transactional. Fencing token makes it safe against zombies.
- **External lock service (ZooKeeper/etcd/Redis):** purpose-built leader election/locks; another dependency + its own
  failure modes (a Redis lock alone isn't safe without fencing).

**Choose:** **DB-based lease + fencing** when the schedule store already gives you atomic conditional updates —
fewer moving parts. Reach for an external lock service mainly for **leader election** across scheduler nodes.

---

## 5. Partitioning vs leader election for "one owner per job"

- **Partition the job space** (hash/bucket → each scheduler owns a shard): no hot-path coordination; must rebalance
  on scaling; brief overlap during rebalance (the atomic claim covers it).
- **Leader election** (one leader per partition, standby fails over): clean HA; adds a coordination dependency.

**Choose:** **partition + atomic claim** as the primary mechanism; add **leader election per partition** for HA
failover. They compose. → [coordination](02-systems/job-scheduler/deep-dives.md#4-preventing-double-fire-across-schedulers-coordination).

---

## 6. Relational schedule store vs bucketed/NoSQL

- **Relational:** `next_run_at` index, `SKIP LOCKED`, atomic conditional claims, transactions — the exactly-once
  primitives for free. Great to a large scale.
- **Bucketed/NoSQL (Redis sorted sets / partitioned KV):** horizontal scale for the due-scan; you re-implement some
  atomicity/claim logic.

**Choose:** **relational until the due-scan outgrows one DB**, then time-bucketed sharding. Run history → cheap append
store regardless. → [data model](02-systems/job-scheduler/api-data-model.md#sql-or-nosql).

---

## 7. Catch-up vs skip missed fires (per job)

- **Catch-up:** run everything that was missed during downtime. Right for "daily report", "billing run".
- **Skip:** drop stale fires, run only the current one. Right for "every-minute health poll" (don't run 120 copies).

**Choose:** **make it per-job policy** with a sensible default (skip stale for high-frequency; catch-up for
low-frequency business jobs). → [cron correctness](02-systems/job-scheduler/deep-dives.md#5-cron--time-correctness-subtle-interviewers-probe-it).

---

## 8. Fixed fire times vs jittered

- **Fixed:** honors the exact cron (`0 * * * *`), but everyone picks round times → **thundering herd** on the hour.
- **Jittered:** spread fires across a window → smooth load; slightly off the exact minute.

**Choose:** **jitter** where the product allows (most background jobs don't care about the exact second) — it's the
difference between a smooth system and a top-of-hour spike. Same principle as [TTL jitter](01-patterns/caching.md#ttl-bound-your-staleness).

---

## The one-paragraph summary (say this)

> *"The core move is separating scheduling from execution: a scheduler finds due jobs via a `next_run_at` index (or
> time buckets at scale), atomically claims each so only one scheduler fires it, then enqueues to a durable queue;
> stateless workers execute. I get exactly-once *effect* — not delivery — via at-least-once enqueue + an idempotency
> ledger + idempotent job logic, and I use leases with **fencing tokens** so a zombie worker that missed its lease
> can't double-apply. Cron jobs recompute their next fire with explicit timezone/DST handling and a per-job
> catch-up-vs-skip policy. I partition jobs across schedulers (with leader election for HA) to avoid double-firing,
> jitter fire times to avoid top-of-hour herds, and retry with backoff into a DLQ. Relational store for the schedule
> (its atomic claims are the exactly-once primitives); cheap append store with TTL for run history."*

→ Next: **[Failure Scenarios](02-systems/job-scheduler/failure-scenarios.md)**
