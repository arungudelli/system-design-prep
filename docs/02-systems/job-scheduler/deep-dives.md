# Distributed Job Scheduler — Deep Dives

The interview lives here: **finding due jobs efficiently**, **exactly-once execution**, **leases + fencing tokens**,
**cron/time correctness**, **retries**, and **long-running jobs**.

---

## 1. Finding what is due, efficiently

The scheduler must answer *"what's due now?"* on every tick without scanning millions of jobs.

- **Index on `next_run_at`:** `SELECT … WHERE next_run_at <= now AND status='SCHEDULED'` reads only the due slice.
  Cost ∝ jobs **due**, not jobs **stored** — the whole trick.
- **Pull disjoint work with `FOR UPDATE SKIP LOCKED`:** multiple schedulers can grab different due rows concurrently
  without blocking each other (each skips rows another scheduler locked). Simple horizontal scaling of the scan.
- **Time bucketing at scale:** group jobs by fire-minute (`bucket = floor(next_run_at/60)`); each tick reads only the
  current bucket(s). Buckets shard cleanly across scheduler nodes and map onto Redis sorted sets (`ZRANGEBYSCORE`)
  or partitioned tables. Handles millions of jobs with tiny per-tick cost.
- **Tick cadence** sets your timing precision: a 1s tick → ~1s accuracy. Tighter precision → faster tick (more load).

> **Interview line:** *"I never scan all jobs. I index by `next_run_at` and read only due rows, pulling disjoint
> batches with `SELECT … FOR UPDATE SKIP LOCKED` so schedulers don't contend. At scale I bucket by fire-minute so
> each tick touches one small bucket and buckets shard across nodes."*

---

## 2. Exactly-once execution (the honest version)

You **cannot** guarantee exactly-once *delivery* (the worker does the work, then must record/ack — a crash in
between means redelivery, see [queues & workers](../../01-patterns/queues-workers.md#delivery-semantics-know-all-three-cold)).
So you engineer **exactly-once *effect*** = **at-least-once + idempotency + a claim/lease with fencing**. Three layers:

**Layer 1 — Don't double-*fire* (scheduler side).**
Only one scheduler should enqueue a given due job. Guarantee it with an **atomic conditional claim**:
```sql
UPDATE jobs SET status='CLAIMED', lease_owner=me, fence_token=fence_token+1
WHERE job_id=? AND status='SCHEDULED';   -- only ONE scheduler's UPDATE matches
```
Only the winner (rows-affected = 1) enqueues. Combined with **partitioning/leader election** so schedulers own
disjoint job sets, this prevents duplicate fires.

**Layer 2 — Don't double-*execute* (worker side, dedup).**
The queue is at-least-once, so a worker may receive a duplicate (redelivery after a lease expiry). Before applying
effects, check the **idempotency ledger** (`job_runs` keyed by `(job_id, scheduled_for)` or a fire id): if this fire
already ran, **skip**. Record the run **in the same transaction as the effect** where possible. See
[idempotency](../../01-patterns/idempotency.md).

**Layer 3 — Make the job itself idempotent.**
The ultimate backstop: design the job's action so running it twice is harmless (upsert, set-absolute-value, use an
idempotency key on the downstream call). For a **payment** job especially, the downstream charge must carry an
idempotency key so a double-execute can't double-charge.

> **Interview line:** *"I can't get exactly-once delivery, so I engineer exactly-once effect: an atomic claim so only
> one scheduler fires the job, a dedup ledger so a redelivered message is skipped, and idempotent job logic as the
> backstop. For a payment I also pass an idempotency key to the charge API."*

---

## 3. Leases & fencing tokens (the zombie-worker problem)

**The problem:** a worker claims a job with a **lease** (visibility timeout). If it's slow/GC-paused/network-
partitioned and misses the lease deadline, the system assumes it's dead and **redelivers** the job to another
worker. But the first worker isn't actually dead — it wakes up and finishes too → **two executions**, and worse,
the zombie might write stale results.

**The fix — fencing tokens:**
- Each claim increments a **monotonic `fence_token`**. The new worker gets a higher token than the zombie.
- Every side-effect / write is tagged with the token; the resource (DB row, downstream) **rejects writes with a
  token lower than the highest it has seen**. The zombie's stale write (old token) is refused.

```text
Worker A claims job, fence=7, starts slow work...
Lease expires → Worker B claims same job, fence=8, executes, writes with fence=8
Worker A wakes, tries to write with fence=7 → REJECTED (7 < 8). No double-apply.
```

This is the classic result from Martin Kleppmann's lock analysis: **a lease/lock alone is not safe** (the holder can
pause past expiry); you need a **fencing token** for correctness. See
distributed locks *(coming)*.

> **Interview line:** *"A lease can be violated by a GC pause or partition — the holder thinks it still owns the job
> after it expired. So I attach a fencing token that increments on each claim; downstream rejects writes with a stale
> token, so a resurrected zombie worker can't double-apply."*

---

## 4. Preventing double-fire across schedulers (coordination)

Two ways to ensure exactly one scheduler owns each job:

- **Partition/shard the job space** (preferred): `partition = hash(job_id) % N` (or by time bucket); each scheduler
  owns some partitions. No two schedulers scan the same jobs → no coordination on the hot path. Rebalance
  partitions on scaling ([sharding](../../01-patterns/sharding.md)).
- **Leader election** (e.g. via ZooKeeper/etcd/Redis lock): one leader per partition does the scheduling; a standby
  takes over on failure. Adds a coordination dependency but gives clean HA.
- **Belt-and-suspenders:** even with partitioning, the **atomic claim** (§2 Layer 1) makes a brief overlap during
  rebalancing safe — two schedulers might both see a job, but only one's conditional UPDATE wins.

---

## 5. Cron & time correctness (subtle, interviewers probe it)

Recurring jobs bring time pitfalls:

- **Compute next run on fire:** after firing, set `next_run_at = nextAfter(cron, now, timezone)`. Store it so the
  index finds it next time.
- **Timezones & DST:** a `0 2 * * *` "America/New_York" job must fire at 2am *local* — which shifts by an hour across
  DST, and on the spring-forward night 2am may **not exist** (skip or run once) and in fall-back may occur **twice**
  (run once). Store the timezone; use a correct calendar library; define the DST policy explicitly.
- **Catch-up vs skip after downtime:** if the scheduler was down from 1am–3am, do the 2am jobs **run late** (catch-up)
  or **skip**? Make it a per-job policy. For "send daily report" catch-up is right; for "every-minute health poll"
  you skip stale fires (don't run 120 backed-up copies).
- **Overlapping runs:** if a job's execution takes longer than its interval (an hourly job that runs 90 min), do you
  allow concurrent runs or **skip/queue** the next? Usually add a "no overlap" option (skip if previous still running).
- **Misfire handling:** define behavior when a fire is late beyond a threshold (run now, run once, or drop).

> **Interview line:** *"For cron I recompute next_run_at on each fire and store the timezone, because DST shifts the
> local fire time and creates skipped/duplicated hours. I make catch-up-vs-skip and overlap policy per-job — a daily
> report should catch up; a health poll should skip stale fires rather than run a hundred backed-up copies."*

---

## 6. Retries, backoff & DLQ

- **Transient failures** (downstream 5xx/timeout) → **retry with exponential backoff + jitter**, capped at
  `max_retries`. Track `attempts` on the job/run.
- **Poison jobs** (fail every time — bad payload, deleted target) → after N attempts move to a **dead-letter queue**
  and alert; don't retry forever and jam a worker. Redrive after fixing.
- **Where retries live:** the durable queue's native retry/visibility-timeout handles worker-side failures; the
  scheduler handles "the fire never got picked up" (lease expiry → re-enqueue).
- **Backoff schedule** should respect the job's nature — a payment retry might be minutes apart with a hard cap; a
  webhook might back off seconds. See [retries/DLQ](../../01-patterns/queues-workers.md#retries-backoff-and-the-dlq).

---

## 7. Long-running jobs

- A job that runs for minutes/hours will **outlive a naive lease** → gets declared dead and re-run. Fix with
  **lease heartbeats:** the worker periodically extends the lease while it's still working. If heartbeats stop
  (worker really died), the lease expires and the job is safely reclaimed (fencing prevents the zombie double-apply).
- Consider **splitting** very long jobs into checkpointed steps so a crash resumes rather than restarts.
- Keep long jobs off the same worker pool as fast jobs if they'd cause head-of-line blocking — separate queues/pools.

---

## 8. At-least-once vs at-most-once (choose per job)

- **At-least-once (+ idempotency)** — the default. Reminder emails, report generation, cache warming: better to run
  twice (deduped) than miss.
- **At-most-once** — when a duplicate is worse than a miss and can't be made idempotent. Rare; achieved by
  ack-before-execute (risking a miss on crash). Usually you avoid needing this by making the job idempotent instead.
- **The payment example:** you want the *effect* exactly once → at-least-once execution + an idempotency key on the
  charge, so retries/duplicates never double-charge and a crash never silently skips.

→ Next: **[Trade-offs](tradeoffs.md)**
