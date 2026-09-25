# Distributed Job Scheduler — Principal-Level Deep Dive (The Stops)

> The **"never be blind"** companion to the base Job Scheduler docs. The scheduler as a **staircase of stops** —
> smallest defensible design first, each bigger stop with its **exact trigger** — ending in the **top-stop
> (max-scale / deliberately over-engineered)** architecture plus a **reconciliation table** for when each heavy
> component is overkill and should be **removed**.
>
> Read the base first: [Requirements](02-systems/job-scheduler/requirements.md) · [Architecture](02-systems/job-scheduler/architecture.md)
> · [Deep Dives](02-systems/job-scheduler/deep-dives.md) · [Trade-offs](02-systems/job-scheduler/tradeoffs.md).
>
> **Say this in an interview:** *"I'll start with a single scheduler polling a `next_run_at` index and grow it one
> bottleneck at a time. I know the sharded, multi-region, sub-second-precision version — but each piece only earns its
> place when a specific ceiling forces it."*

---

## The stops (climb one lever at a time)

| Stop | Scale / SLA target | Shape | **Trigger that forces the NEXT stop** |
|---|---|---|---|
| **1 · Cron on a box** | thousands of jobs, minute precision | one process: DB table + `next_run_at` index + in-process executor | executor can't keep up; a crash misses fires; no retries |
| **2 · Scheduler + queue + workers** | ~100s–1k fires/s, 99.9% | scheduler claims due jobs → **durable queue** → stateless workers; retries → DLQ; **lease + idempotency** | due-scan/claim contention; thundering herd; need HA |
| **3 · Partitioned + HA (base target)** | ~10k fires/s, 99.99% | **partitioned schedulers** (`SKIP LOCKED`, leader election), **fencing tokens**, jittered fires, per-tenant limits | single DB can't hold the due-scan; multi-region; sub-second precision |
| **4 · Sharded, multi-region, precise (top stop)** | ~100k+ fires/s, 99.999%, sub-second | **time-bucketed sharded** schedule store + **hierarchical timing wheels** for precision + **multi-region** schedulers + external coordination (etcd/ZK) + per-tenant fairness/quotas | (ceiling of a general scheduler; DAGs → a workflow engine) |

> Most interviews want **Stop 2–3**. Reach into Stop 4 only when the prompt says "millions of jobs / sub-second /
> multi-region / strict fairness." Name the trigger each time.

---

## Top-stop architecture (max-scale)

```text
  API → Schedule Store: TIME-BUCKETED + SHARDED (bucket = floor(next_run_at/second|minute), sharded by bucket/hash)
          ▲   each shard replicated (primary+replicas), multi-region
          │
   SCHEDULER FLEET (partitioned; leader election per partition via etcd/ZooKeeper)
     - each owns disjoint buckets → scan only its current bucket(s)
     - HIERARCHICAL TIMING WHEELS in memory for sub-second precision (loaded from the durable bucket)
     - atomic claim (fence++) → enqueue
          │ enqueue (at-least-once)
          ▼
     Durable Queue (per-priority / per-tenant lanes)  ── Kafka/partitioned queue
          │
     WORKER FLEET (stateless, autoscaled, multi-region)
     - lease + fencing token · dedup ledger · idempotent execution · heartbeat long jobs
     - retries+backoff → DLQ ; write job_runs → cheap append store (TTL)
          │
     Per-tenant rate limiting + fairness scheduler (no tenant starves others)
     Telemetry → Kafka → stream processing → dashboards (fire latency, misfires, lag)
```

---

## Deep dives that only matter at the top stop

### Time-bucketed sharded schedule store (Stop 3 → 4 trigger: **due-scan outgrows one DB**)
- **Why:** at ~100k fires/s a single indexed table's due-scan + claim contention becomes the ceiling.
- **Design:** bucket jobs by fire-time (`bucket = floor(next_run_at / granularity)`); shard buckets across nodes;
  each scheduler scans only **its** current bucket(s) → tiny, bounded per-tick work that scales horizontally. Maps
  onto Redis sorted sets per bucket or partitioned tables. See [finding due jobs](02-systems/job-scheduler/deep-dives.md#1-finding-what-is-due-efficiently).
- **Over-engineering watch:** *only when the indexed-column + `SKIP LOCKED` approach can't keep up.* Below ~tens of
  thousands of fires/s, a single indexed store is simpler and correct; **remove** bucketing.

### Hierarchical timing wheels (trigger: **sub-second precision at scale**)
- **Why:** polling on a 1s tick gives ~1s accuracy. Sub-second precision by polling faster hammers the store.
- **Design:** poll the durable store to load the next time-window into an **in-memory hierarchical timing wheel**
  (the data structure Kafka/Netty use for millions of timers) → O(1) fire scheduling at high precision, backed by the
  durable bucket store for crash-safety.
- **Over-engineering watch:** *only when the product needs sub-second fires.* Most schedulers are fine at second/
  minute precision; **remove** the timing wheel and just poll.

### Multi-region schedulers (trigger: **99.999% + global jobs**)
- **Why:** a single-region scheduler is a regional SPOF; five-nines can't tolerate a region loss.
- **Design:** partition the job space across regions (a job is owned by one region's scheduler); durable, replicated
  store means another region can take over a dead region's partitions; the **atomic claim** keeps a brief overlap safe.
- **Hard part:** avoid two regions firing the same job → partition ownership + claim + leader election (etcd/ZK).
- **Over-engineering watch:** *only for five-nines / global.* Single-region multi-AZ hits 99.99%; **remove**
  multi-region below that.

### Per-tenant fairness & quotas (trigger: **multi-tenant, one tenant floods**)
- **Why:** a tenant scheduling 10M jobs at once could starve everyone.
- **Design:** per-tenant [rate limiting](01-patterns/rate-limiting.md) + fair-scheduling (weighted lanes) so each
  tenant gets a bounded share; separate transactional-vs-bulk lanes; jitter to kill top-of-hour herds.
- **Over-engineering watch:** *only when multi-tenant abuse is real.* Single-tenant / trusted internal use doesn't
  need it; **remove** the fairness layer.

### External coordination (etcd/ZooKeeper) (trigger: **leader election across a scheduler fleet**)
- **Why:** electing one owner per partition across many scheduler nodes needs strong coordination.
- **Design:** etcd/ZooKeeper for leader election + partition assignment; the **DB atomic claim** still does per-job
  ownership (fewer moving parts than a lock service per job).
- **Over-engineering watch:** *only for a multi-node scheduler fleet.* A single active scheduler (with a hot standby)
  needs no coordination service; **remove** it.

---

## Reconciliation table — what forces each heavy component (and when to remove it)

| Component | Base docs (Stop 2–3, ~10k fires/s) | Top stop (Stop 4) | Trigger to ADOPT | When it's OVERKILL → remove |
|---|---|---|---|---|
| Schedule store | indexed `next_run_at` + `SKIP LOCKED` | time-bucketed + sharded | due-scan/claim outgrows one DB | ≤ ~tens of k fires/s |
| Timing precision | poll on a tick (~1s) | hierarchical timing wheels | product needs sub-second fires | second/minute precision is fine |
| Regions | single-region multi-AZ | multi-region schedulers | 99.999% / global jobs | 99.99% target |
| Coordination | atomic DB claim + hot standby | etcd/ZooKeeper leader election | multi-node scheduler fleet | single active scheduler |
| Fairness | per-tenant limits (basic) | weighted fair scheduling + quotas | multi-tenant abuse/starvation | single/trusted tenant |
| Analytics | audit rows / metrics | Kafka → stream processing | real-time ops at scale | simple metrics suffice |
| Queue | durable queue (SQS/RabbitMQ) | Kafka partitioned lanes | replay / very high fan-out | plain task queue is enough |

> Lead with Stop 2–3. When pushed: *"for millions of jobs at sub-second precision across regions with strict
> fairness, here's Stop 4 — and each component maps to the ceiling that forces it. Below that ceiling I'd remove it."*
> And always note: **if jobs gain dependencies (DAGs), that's a workflow engine (Airflow/Step Functions), not this.**

---

## The 60-second Principal summary

> *"Core move: separate scheduling from execution. I start with one scheduler polling a `next_run_at` index, claiming
> due jobs atomically and enqueueing to a durable queue for stateless workers — with leases, fencing tokens, idempotent
> execution, retries→DLQ, and jittered fires. That's ~10k fires/s at four nines. Pushed higher, I climb one lever at a
> time: time-bucketed sharded store when the due-scan outgrows one DB, hierarchical timing wheels for sub-second
> precision, multi-region schedulers (partition ownership + leader election) for five-nines, and weighted per-tenant
> fairness when a tenant could starve others. Each has a named trigger and I'd remove it below that trigger. And if
> jobs need dependencies, I'd reach for a workflow engine, not extend this scheduler."*

← Back to **[Job Scheduler overview](02-systems/job-scheduler/README.md)** · Base **[deep dives](02-systems/job-scheduler/deep-dives.md)** · **[cheatsheet](02-systems/job-scheduler/cheatsheet.md)**
