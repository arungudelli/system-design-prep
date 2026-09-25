# Distributed Job Scheduler — Interview Questions

> Answer each **out loud first**, then expand and self-critique against the
> [critique lens](00-framework/staff-level-thinking.md#how-your-answer-gets-graded-the-critique-lens).

---

## Core

**Q1. What's the single most important architectural decision here?**
<details><summary>Model answer</summary>

**Separate scheduling from execution.** A scheduler decides *what's due now* and enqueues it; a worker pool pulls
from a durable queue and *executes*. This lets each scale and fail independently, buffers bursts in the queue, and
keeps the hard time+coordination logic in the scheduler while execution stays simple/stateless. Everything else
follows from this boundary. See [architecture](02-systems/job-scheduler/architecture.md#the-core-insight-separate-scheduling-from-execution).
</details>

**Q2. How do you find which jobs are due without scanning millions every second?**
<details><summary>Model answer</summary>

Keep a **time-ordered index on `next_run_at`** and query `WHERE next_run_at <= now AND status='SCHEDULED'` — cost is
proportional to jobs *due*, not jobs *stored*. Pull disjoint batches with `SELECT … FOR UPDATE SKIP LOCKED` so many
schedulers grab different rows without contention. At scale, **time-bucket** jobs by fire-minute so each tick reads a
tiny bucket that shards across nodes. See [finding due jobs](02-systems/job-scheduler/deep-dives.md#1-finding-what-is-due-efficiently).
</details>

**Q3. How do you guarantee a job runs exactly once?**
<details><summary>Model answer</summary>

You **can't** guarantee exactly-once *delivery*; you engineer exactly-once *effect* in three layers: (1) an **atomic
conditional claim** so only one scheduler fires the job; (2) an **idempotency ledger** so a redelivered message is
skipped; (3) **idempotent job logic** (upsert / idempotency key on the downstream) as the backstop. Plus **fencing
tokens** so a zombie worker can't double-apply. See [exactly-once](02-systems/job-scheduler/deep-dives.md#2-exactly-once-execution-the-honest-version).
</details>

**Q4. Two workers end up running the same job. How is that possible, and how do you make it safe?**
<details><summary>Model answer</summary>

Possible because the queue is at-least-once: a slow worker misses its lease/visibility timeout, the job is
redelivered, and the original later resumes (a **zombie**). Make it safe with **idempotency** (dedup ledger + the job
being idempotent) and **fencing tokens** — each claim increments a monotonic token; the resource rejects writes with
a stale token, so the zombie's late write is refused. A lease alone isn't safe; fencing makes it correct.
</details>

---

## Coordination & time

**Q5. How do you stop two schedulers from firing the same job?**
<details><summary>Model answer</summary>

**Partition the job space** (hash(job_id) or time bucket) so each scheduler owns disjoint jobs, with **leader
election per partition** for HA. As a safety net, the **atomic conditional claim** (`UPDATE … WHERE
status='SCHEDULED'`) means even if two schedulers see the same row during a rebalance, only one's update wins and
enqueues. See [coordination](02-systems/job-scheduler/deep-dives.md#4-preventing-double-fire-across-schedulers-coordination).
</details>

**Q6. What's a fencing token and why do you need it if you already have a lease?**
<details><summary>Model answer</summary>

A lease can be *violated* — a GC pause or network partition can make the holder run past the lease's expiry while the
system has already reassigned the job. So a lease alone allows two concurrent holders. A **fencing token** is a
monotonic number incremented on each claim; every write carries it, and the resource rejects any write with a token
lower than the highest seen. The zombie (old token) is fenced out. Correctness, not just mutual exclusion.
</details>

**Q7. The scheduler was down from 1am to 3am. What happens to the 2am jobs?**
<details><summary>Model answer</summary>

Because `next_run_at` lives in a **durable store**, on recovery the scheduler queries overdue jobs and applies a
**per-job catch-up-vs-skip policy**: business jobs (daily report, billing) **run late (catch-up)**; high-frequency
jobs (every-minute poll) **skip stale fires** so you don't run 120 backed-up copies. HA via leader-election standby
minimizes the outage in the first place. See [failures](02-systems/job-scheduler/failure-scenarios.md#scheduler-crashes--is-down-at-a-fire-time).
</details>

**Q8. How do you handle cron jobs across timezones and DST?**
<details><summary>Model answer</summary>

Store the **timezone** with the cron expression and compute `next_run_at` in local time using a correct calendar
library. DST is the trap: `0 2 * * *` local shifts by an hour, and on spring-forward 2am **doesn't exist** (skip or
run once) while fall-back 2am **occurs twice** (run once). Define the DST policy explicitly. Recompute next run on
each fire and store it. See [cron correctness](02-systems/job-scheduler/deep-dives.md#5-cron--time-correctness-subtle-interviewers-probe-it).
</details>

---

## Robustness & scale

**Q9. Everyone schedules jobs for the top of the hour. What breaks and what do you do?**
<details><summary>Model answer</summary>

A **thundering herd** at :00 — millions of jobs fire simultaneously and overwhelm workers/downstreams. **Jitter** the
fire/execution times across the window, let the **queue buffer** the burst, and apply **per-tenant rate limiting** so
one tenant's mass schedule can't starve others. Same jitter principle as cache TTLs.
</details>

**Q10. A job runs for 2 hours. How do you keep it from being re-run as "dead"?**
<details><summary>Model answer</summary>

**Lease heartbeats:** the worker periodically extends its lease while working, so the job isn't declared dead and
reassigned. If heartbeats stop (worker truly died), the lease expires and it's safely reclaimed — with **fencing** to
stop the original from double-applying if it comes back. Consider checkpointing long jobs so a crash resumes rather
than restarts. See [long jobs](02-systems/job-scheduler/deep-dives.md#7-long-running-jobs).
</details>

**Q11. A job's webhook target starts timing out. What happens?**
<details><summary>Model answer</summary>

Per-execution **timeout**; **retry with exponential backoff + jitter**, capped at `max_retries`; then **DLQ + alert**.
Wrap the call in a **circuit breaker** + bulkhead so a broadly-failing target doesn't exhaust the worker pool. Never
retry-storm a struggling downstream. Fixed poison jobs get **redriven** from the DLQ.
</details>

**Q12. What's the first bottleneck as you scale, and how do you address it?**
<details><summary>Model answer</summary>

The **due-scan on the schedule store** (every tick queries "what's due"). Address with the `next_run_at` index, then
**time-bucketing** and **partitioning the job space across schedulers** (with `SKIP LOCKED` to avoid contention).
Execution throughput scales separately by adding stateless workers; the queue decouples fire rate from execution
rate. Run-history writes are the storage driver → TTL + cheap append store.
</details>

---

## Staff-level curveballs

**Q13. Is it worse to run a job twice or to miss it? How does that change the design?**
<details><summary>Model answer</summary>

Depends on the job — and you should ask. A **reminder email**: better twice (deduped) than never → at-least-once +
idempotency. A **payment charge**: never twice → still at-least-once execution but with an **idempotency key on the
charge API** so duplicates are absorbed, and never silently skipped. The guarantee is chosen per-job; the machinery
(claim, ledger, fencing, idempotent action) supports both by making duplicates harmless and drops impossible.
</details>

**Q14. Would you poll for due jobs or use timers? Why?**
<details><summary>Model answer</summary>

**Poll a durable due-index** for a distributed scheduler — it survives crashes (timers live in memory and are lost on
restart) and is simple to shard. In-memory **timer wheels** give sub-second precision but aren't durable/distributed
on their own. A good hybrid: poll to load the next time-window into an in-memory timer wheel for precision, backed by
the durable store. See [poll vs push](02-systems/job-scheduler/tradeoffs.md#1-poll-pull-vs-push-scheduling).
</details>

**Q15. Where would you use a distributed lock vs a database claim?**
<details><summary>Model answer</summary>

Prefer the **DB atomic conditional claim** for owning a *job* — the schedule store already gives transactional
compare-and-set, no extra infra, and fencing tokens keep it safe. Use an **external lock service (ZooKeeper/etcd)**
mainly for **leader election** across scheduler nodes (who owns which partition). Remember a Redis/lock alone isn't
safe without a fencing token.
</details>

**Q16. How does this system reuse the patterns you've studied?**
<details><summary>Model answer</summary>

**Queue + workers:** decouples scheduling from execution; visibility timeouts = leases. **Idempotency:** the whole
exactly-once-effect story (ledger + idempotent jobs). **Sharding:** partition the job space / time buckets across
schedulers. **Rate limiting:** per-tenant fairness + herd control. **Distributed locks/fencing:** leader election +
zombie protection. Naming this composition shows you see the system as patterns assembled, not a bespoke design.
</details>

**Q17. What changes at 10× and 100×?**
<details><summary>Model answer</summary>

**10×:** more scheduler partitions + workers; jitter harder; watch due-scan index hot-spotting on "now". **100×:**
**time-bucketed sharded** schedule store (single DB can't hold the due-scan), multi-region schedulers, strict
per-tenant quotas/fairness, and run-history offloaded to object storage with aggressive TTL. The scheduler/queue/
worker split and the exactly-once-effect machinery stay the backbone.
</details>

**Q18. When would you redesign this system?**
<details><summary>Model answer</summary>

When requirements break a core assumption: jobs gain **dependencies/DAGs** (move to a workflow engine like
Airflow/Step Functions), timing needs go **sub-millisecond** (dedicated timer infrastructure), you must run
**untrusted code** (a sandboxed execution platform), or scale jumps 100× (bucketed multi-region sharding). Absent
those, the separation-of-concerns backbone holds.
</details>

---

## Self-scoring

Grade against the [15-point critique lens](00-framework/staff-level-thinking.md#how-your-answer-gets-graded-the-critique-lens):
did you separate scheduling from execution, index the due-scan, give the honest exactly-once-effect answer, use
leases + **fencing**, handle scheduler-down catch-up + cron/DST, and connect it to the patterns? Note your weakest
area and drill it.

← Back to **[README](02-systems/job-scheduler/README.md)** · Next: **[Cheatsheet](02-systems/job-scheduler/cheatsheet.md)**
