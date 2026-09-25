# Distributed Job Scheduler — Architecture

## The core insight: separate *scheduling* from *execution*

The single most important design decision. Two distinct responsibilities, decoupled by a queue:

- **Scheduler** — answers *"what's due now?"*, claims each due job, and **enqueues** it. Cares about **time**.
- **Worker pool** — pulls from the queue and **executes**. Cares about **doing the work**.

Decoupling means: they **scale independently** (a burst of fires doesn't need instant execution — the queue
buffers), **fail independently**, and the hard "time + coordination" logic stays in one place (the scheduler) while
execution stays simple and horizontally scalable.

---

## 9. Architecture

```text
  Clients ──POST /jobs──►  [ API ]  ──► Schedule Store (jobs, next_run_at indexed)
                                              ▲   │
                                              │   │ (1) tick: SELECT due jobs (next_run_at <= now)
                                              │   ▼
                                     ┌──────────────────────┐
                                     │   SCHEDULER(S)        │  partitioned/leader-elected
                                     │  - scan due jobs      │  so each job is owned by ONE scheduler
                                     │  - atomically CLAIM   │  (2) claim: UPDATE ... WHERE status=SCHEDULED
                                     │  - enqueue to queue   │  (3) recompute next_run_at (cron) / mark done (once)
                                     └───────────┬──────────┘
                                                 │ (4) enqueue job (at-least-once)
                                                 ▼
                                         [ Durable Queue ]  (SQS/Kafka/…)
                                                 │
                          ┌──────────────────────┼──────────────────────┐
                          ▼                       ▼                       ▼
                    [ Worker ]              [ Worker ]               [ Worker ]   (stateless, autoscaled)
                     - lease + fence          - execute (idempotent)    - ack on success
                     - heartbeat long jobs     - retry/backoff → DLQ    - write job_runs history
```

**Every component justified:**
- **API** — CRUD for jobs; writes to the schedule store.
- **Schedule store** — durable jobs table with the `next_run_at` index (see [data model](api-data-model.md)).
- **Scheduler(s)** — the time brain: find due → **claim atomically** → enqueue → advance `next_run_at`. Partitioned
  or leader-elected so **exactly one scheduler owns each job** (no double-fire).
- **Durable queue** — decouples scheduling from execution; buffers bursts; gives retries/DLQ/visibility-timeout
  (a [queue + workers](../../01-patterns/queues-workers.md)).
- **Workers** — stateless executors; lease each job with a **fencing token**, run idempotently, retry, record history.

---

## 10. The fire loop (step by step)

```text
Every tick (e.g. 1s), each scheduler partition:
1. SELECT jobs WHERE next_run_at <= now AND status='SCHEDULED'
        FOR UPDATE SKIP LOCKED LIMIT N          -- grab a disjoint batch
2. For each due job, ATOMICALLY claim it:
        UPDATE jobs SET status='CLAIMED', lease_owner=me, fence_token=fence_token+1
        WHERE job_id=? AND status='SCHEDULED'   -- only one claimer wins
3. Enqueue {job_id, fence_token, scheduled_for} to the durable queue (at-least-once)
4. Advance the schedule:
        - ONCE  → status stays CLAIMED→(executed); no next run
        - CRON  → compute next_run_at from cron+timezone; set status back to SCHEDULED, next_run_at=next
```

```text
Worker side:
5. Pull job from queue (visibility timeout starts)
6. Check idempotency ledger (job_runs): already ran this fire? → skip (duplicate)
7. Execute the job (webhook/task); for long jobs, HEARTBEAT to extend the lease
8. On success → write job_runs(SUCCEEDED), ack (delete from queue)
   On failure → retry w/ backoff; after max_retries → DLQ + mark FAILED + alert
```

- **Why claim *before* enqueue:** so two schedulers can't both enqueue the same job. The atomic conditional update
  (step 2) means exactly one scheduler wins the claim.
- **Why fence_token:** if a slow/paused worker resumes after its lease was reassigned, its writes carry a **stale
  token** and are rejected — prevents a zombie worker from double-applying. See
  [exactly-once deep dive](deep-dives.md#2-exactly-once-execution-the-honest-version).

---

## 11. Bottlenecks (ranked)

| Rank | Bottleneck | Why | Detect via | Fix |
|---|---|---|---|---|
| 1 | **Due-scan on schedule store** | every tick queries "what's due" | scan latency, DB CPU at each tick | **index `next_run_at`** / **time-bucket** / shard |
| 2 | **Scheduler throughput / contention** | claiming many due jobs/tick | claim latency, lock contention | `SKIP LOCKED`, **partition jobs across schedulers** |
| 3 | **Execution throughput** | 10k fires/sec to run | queue depth, consumer lag | add **stateless workers**; queue buffers bursts |
| 4 | **Thundering herd at round times** | everyone schedules "0 * * * *" (top of hour) | spike at :00 | **jitter** fire times; spread within the window |
| 5 | **Run-history writes** | huge append volume | write latency, storage growth | append store + **TTL/retention** |

**The classic real-world gotcha (#4):** a huge fraction of cron jobs are set to `0 * * * *` or midnight → massive
synchronized spikes. Add **jitter** to fire times (or spread execution across the minute) so you don't get a
thundering herd on the hour. (Same idea as [TTL jitter in caching](../../01-patterns/caching.md#ttl-bound-your-staleness).)

---

## 12. Scaling each bottleneck

- **Due-scan** → index `next_run_at`; at scale, **time-bucket** jobs and shard buckets across scheduler nodes so
  each scans a small slice.
- **Scheduler** → **partition the job space** (e.g. by hash(job_id) or by bucket) so each scheduler owns a shard and
  they don't contend; `SELECT … FOR UPDATE SKIP LOCKED` lets even multiple schedulers on one shard grab disjoint
  rows. Use **leader election** per partition for HA (a standby takes over if the owner dies).
- **Execution** → add stateless workers; the **queue decouples** fire rate from execution rate so bursts buffer.
- **Herd** → jitter fire times; rate-limit per-tenant so one tenant's 1M-job burst doesn't starve others
  ([rate limiting](../../01-patterns/rate-limiting.md)).
- **History** → cheap append store + TTL; sample/aggregate if full history isn't needed.

---

## 20. Evolution with scale

| Stage | Shape | What forced the jump |
|---|---|---|
| **1 · Small** | one process: DB table + `next_run_at` index + in-process executor | — |
| **2 · Moderate** | scheduler + **durable queue** + worker pool; claim-before-enqueue | executor couldn't keep up; needed crash-safety + retries |
| **3 · Large** | **partitioned schedulers** (leader election), `SKIP LOCKED`, jittered fires, DLQ | due-scan/claim contention; thundering herd; HA |
| **4 · Extreme** | **time-bucketed sharded** schedule store, multi-region, per-tenant fairness/quotas | single DB can't hold the due-scan; multi-tenant scale |

> The transitions are driven by **due-scan load and coordination**, not raw throughput — correctness under crashes
> and time is the recurring theme.

→ Next: **[Deep Dives](deep-dives.md)**
