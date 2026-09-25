# Distributed Job Scheduler — Failure Scenarios

> The recurring theme: **never silently drop a due job**, **never double-apply**, and stay correct despite crashes,
> pauses, clock skew, and downtime. The two nightmares are a **missed fire** (job never ran) and a **double execution**
> (job ran twice with real side effects).

---

## Worker crashes mid-execution

- **Impact:** the job was picked up but not finished/acked.
- **Detect:** lease/visibility-timeout expiry; missing heartbeat.
- **Contain/recover:** the queue **redelivers** on visibility-timeout expiry → another worker runs it. A duplicate is
  harmless because of the **idempotency ledger + fencing token** (a resurrected original can't double-apply). Workers
  are stateless → just replace.
- **Long jobs:** the worker **heartbeats to extend the lease** while working, so a still-alive long job isn't wrongly
  reclaimed.

## Scheduler crashes / is down at a fire time

- **Impact:** jobs due during the outage don't fire on time — the classic **missed fire**.
- **Detect:** scheduler heartbeat / no fires happening.
- **Contain/recover:** because due state is in a **durable store** (not memory), when the scheduler recovers it
  queries `next_run_at <= now` and finds the overdue jobs. Then apply the **catch-up-vs-skip policy** per job (run
  late vs drop stale — see [cron correctness](deep-dives.md#5-cron--time-correctness-subtle-interviewers-probe-it)).
  **HA:** partition + **leader election** so a standby scheduler takes over the dead one's partitions quickly.
- **This is why the schedule + `next_run_at` live in durable storage** — an in-memory scheduler would lose the schedule.

## Two schedulers fire the same job (double-fire)

- **Impact:** the job gets enqueued twice → risk of double execution.
- **Cause:** overlap during partition rebalancing, or a leader-election hiccup (two think they're leader).
- **Contain:** the **atomic conditional claim** (`UPDATE … WHERE status='SCHEDULED'`) means only one scheduler's
  update matches → only one enqueues. Even if both scan the row, only one wins. Belt-and-suspenders over partitioning.

## Zombie worker (GC pause / network partition past lease)

- **Impact:** a worker thought dead (lease expired, job reassigned) wakes up and completes → double execution + stale
  writes.
- **Contain:** **fencing tokens** — the reassigned worker holds a higher token; the resource rejects the zombie's
  stale-token writes. A lease alone is *not* safe; fencing is what makes it correct. See
  [leases & fencing](deep-dives.md#3-leases--fencing-tokens-the-zombie-worker-problem).

## Downstream target is slow or failing (webhook 5xx / timeout)

- **Impact:** job executions fail or hang.
- **Contain:** per-execution **timeout**; **retry with backoff + jitter**, capped; after `max_retries` → **DLQ + alert**.
  Wrap the downstream call in a **circuit breaker** so a broadly-failing target doesn't tie up the whole worker pool;
  isolate with bulkheads. Never retry-storm a struggling downstream.

## Poison job (fails every attempt)

- **Impact:** a malformed/impossible job retries forever, wasting workers.
- **Contain:** cap retries → **dead-letter** it + alert; **redrive** after fixing the payload/target. A poison job must
  never block the queue for healthy jobs.

## Thundering herd at round times

- **Impact:** millions of `0 * * * *` jobs fire at :00 → spike overwhelms workers/downstreams.
- **Detect:** periodic load spikes aligned to the clock.
- **Contain:** **jitter** fire/execution times across the window; **per-tenant rate limiting** so one tenant's mass
  schedule can't starve others; the queue buffers the burst so execution smooths out.

## Schedule store (DB) fails / fails over

- **Impact:** can't read due jobs or claim them → firing pauses; a **missed fire** window.
- **Contain:** replicate the store (primary + replicas), automated failover; after failover the durable `next_run_at`
  index still holds all due jobs → catch-up runs them. Firing resumes; no schedule lost.

## Queue unavailable

- **Impact:** scheduler can't enqueue claimed jobs.
- **Contain:** the job stays **CLAIMED** in the store (not lost); on queue recovery, re-enqueue. Or the lease on the
  claim expires and it's re-claimed and enqueued later. Durable state means the fire is delayed, not dropped.

## Clock skew across nodes

- **Impact:** schedulers/workers disagree on "now" → early/late fires, or lease timing errors.
- **Contain:** sync clocks (NTP); base fire decisions on the **store's** timestamps where possible; make lease
  timeouts generous relative to expected skew; fencing tokens don't depend on wall-clock, so double-apply protection
  holds even under skew.

## Full restart / deploy

- **Impact:** everything pauses briefly.
- **Contain:** all state is durable (schedule + claims + history) → resume by re-scanning the due-index. Rolling
  deploys so only some schedulers/workers restart at once; leader election re-assigns partitions; leases cover
  in-flight jobs of restarting workers.

---

## Failure-handling toolkit used here

`durable schedule + next_run_at index` (survive scheduler down → catch-up) · `atomic conditional claim` (no double-
fire) · `lease + heartbeat` (worker crash → redeliver; long jobs stay alive) · `fencing token` (no zombie double-
apply) · `idempotency ledger + idempotent jobs` (exactly-once effect) · `retry+backoff+jitter → DLQ` · `circuit
breaker/bulkhead` (bad downstream) · `jitter + per-tenant limits` (herd) · `leader election + partitioning` (HA, one
owner per job) · `NTP + store-based time` (skew).

## Priorities (say this)

> *"My guarantees are: never silently drop a due job, and never double-apply side effects. Durability of the schedule
> gives the first — a downed scheduler catches up from the `next_run_at` index on recovery. Idempotency plus fencing
> tokens give the second — a redelivered or zombie execution is deduped or rejected. The atomic claim and
> partitioning stop two schedulers from firing the same job. Everything degrades to *delayed*, never *lost*."*

→ Next: **[Interview Questions](interview-questions.md)**
