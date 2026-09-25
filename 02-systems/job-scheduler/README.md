# Distributed Job Scheduler

> Design a system that runs jobs — **at a specific time** (run this at 2am), **on a schedule** (every hour / cron),
> or **as soon as possible** (background work) — reliably, at scale, across a fleet of workers. Think Cron-as-a-
> service, AWS EventBridge Scheduler, Quartz, Airflow's scheduler, or the thing that sends "your subscription
> renews tomorrow" emails.

This is the system where **time**, **exactly-once execution**, and **distributed coordination** collide. The
signal is in: how you find "what's due now" efficiently at scale, how you guarantee a job runs **once** (not zero
times, not twice) despite crashes, how you avoid **two schedulers/workers firing the same job**, and how you handle
retries, long-running jobs, and failures — all built on the [queues + workers](01-patterns/queues-workers.md),
[idempotency](01-patterns/idempotency.md), and [distributed locks](01-patterns/distributed-locks.md) *(coming)* you've studied.

---

## Why it's a great system to study

- **Separates scheduling from execution** — a clean architectural boundary (the scheduler decides *when*; workers
  do *what*). Nailing this decoupling is the core insight.
- **"Exactly-once" the honest way** — you can't get exactly-once *delivery*, so you get exactly-once *effect* via
  at-least-once + idempotency + leases/fencing. This system makes that concrete.
- **Time is hard** — clock skew, "what if the scheduler was down at the fire time?", missed vs duplicate fires,
  timezones/DST for cron.
- **Coordination** — leader election / partitioning so one (and only one) component owns each job's scheduling.
- **Two workloads in one** — recurring (cron) and one-off (run-at) jobs, plus immediate background jobs.

---

## Read in this order

1. **[Requirements](02-systems/job-scheduler/requirements.md)** — problem, clarifying questions, functional + NFRs
2. **[Capacity](02-systems/job-scheduler/capacity.md)** — jobs/sec, due-scan rate, storage
3. **[API & Data Model](02-systems/job-scheduler/api-data-model.md)** — job/schedule/run schemas + the "due index"
4. **[Architecture](02-systems/job-scheduler/architecture.md)** — scheduler + queue + workers, the fire loop, scaling
5. **[Deep Dives](02-systems/job-scheduler/deep-dives.md)** — finding due jobs, exactly-once, leases + fencing, cron, retries, long jobs
6. **[Trade-offs](02-systems/job-scheduler/tradeoffs.md)** — the decisions, both sides
7. **[Failure Scenarios](02-systems/job-scheduler/failure-scenarios.md)** — scheduler down at fire time, worker crash, dup fire
8. **[Interview Questions](02-systems/job-scheduler/interview-questions.md)** — attempt first, then reveal
9. **[Cheatsheet](02-systems/job-scheduler/cheatsheet.md)** — 1-page revision
10. **[★ Principal Deep Dive](02-systems/job-scheduler/principal-deep-dive.md)** — the **staged stops** (cron-on-a-box → sharded multi-region) + max-scale variant (time-bucketed sharding, timing wheels, multi-region, fairness) with a reconciliation table of when to adopt/remove each piece

---

## The 30-second version (know this cold)

- **Separate the two concerns:** a **Scheduler** figures out *what's due now* and enqueues it; a **worker pool**
  pulls from a queue and *executes*. Decoupling = each scales independently and failures isolate.
- **Finding due jobs:** don't scan all jobs — keep a **time-ordered index** (DB index on `next_run_at`, or bucketed
  by minute) and query `WHERE next_run_at <= now`. The scheduler polls this on a short tick.
- **Exactly-once execution (the honest version):** at-least-once enqueue + **idempotent job execution** + a
  **claim/lease with a fencing token** so a duplicate or a resurrected worker can't double-apply. You aim for
  exactly-once *effect*, not exactly-once *delivery*.
- **Don't double-fire:** only one scheduler should fire a given job → **partition jobs across schedulers** (shard)
  or use **leader election**; claim each due job with an atomic conditional update before enqueuing.
- **Recurring jobs (cron):** on fire, compute the **next** `next_run_at` and store it (careful with timezones/DST,
  overlapping runs, and catch-up after downtime).
- **Retries + DLQ:** failed jobs retry with backoff; poison jobs go to a **dead-letter** for inspection.
- **Bottlenecks:** the due-scan on the schedule store (index it / bucket it), and execution throughput (scale workers).
