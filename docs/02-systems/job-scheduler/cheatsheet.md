# Distributed Job Scheduler — Cheatsheet

> One-page revision. Reproduce from memory and you can drive the interview.

---

## Problem
Run jobs at a time (run-at), on a schedule (cron), or ASAP — reliably, at scale, across workers, with retries.

## Profile (say first)
**Coordination + reliability, moderate throughput.** The hard part is **correctness under crashes + time**, not QPS.

## Requirements
- **Must:** schedule one-off + recurring, execute near time reliably (at-least-once), retries→DLQ, cancel/query.
- **Defer:** DAGs/dependencies, sandboxed arbitrary code, priorities/fairness, backfill.
- **Dominant NFRs:** reliable near-on-time execution · **exactly-once effect** · **durable schedule**.

## THE core insight
**Separate scheduling from execution.** Scheduler decides *when* → enqueues; workers *execute*. Decoupled = scale +
fail independently, queue buffers bursts.

## Numbers (example)
| | |
|---|---|
| Stored jobs | 100M (~100 GB) — not the bottleneck |
| Peak fires | ~10k/s → queue + stateless workers |
| Due-scan | can't scan 100M/tick → **index `next_run_at`** / bucket |
| Run history | ~430 GB/day → **TTL + append store** (size driver) |

## Architecture
```text
API → Schedule Store (jobs, next_run_at INDEXED)
          ▲
     SCHEDULER(s)  (partitioned / leader-elected)
       1 scan due (next_run_at<=now)
       2 ATOMIC claim (UPDATE WHERE status=SCHEDULED, fence++)
       3 enqueue → Durable Queue
       4 cron: recompute next_run_at
                 │
            Workers (stateless): lease+fence → dedup ledger → execute idempotently
                    → ack / retry+backoff → DLQ ; write job_runs ; heartbeat long jobs
```

## Finding due jobs
Index `next_run_at`; `SELECT … WHERE next_run_at<=now FOR UPDATE SKIP LOCKED LIMIT N` (disjoint, no contention).
At scale: **time-bucket** by fire-minute; buckets shard across schedulers. Cost ∝ jobs *due*, not *stored*.

## Exactly-once EFFECT (not delivery)
1. **Atomic claim** → only one scheduler fires.
2. **Idempotency ledger** (job_runs by job_id+scheduled_for) → skip duplicates.
3. **Idempotent job** (upsert / idempotency key on downstream) → backstop.

## Leases + fencing (zombie problem)
Lease = visibility timeout; **long jobs heartbeat** to extend. A GC-paused worker can outlive its lease →
**fencing token** (monotonic, bumped per claim); resource rejects stale-token writes → no double-apply.
*A lease alone is NOT safe.*

## No double-fire across schedulers
**Partition** job space (hash/bucket) + **leader election** (HA) + **atomic claim** as safety net.

## Cron / time correctness
Recompute next_run_at on fire · store **timezone** · **DST** = 2am may not exist / occur twice · **catch-up vs skip**
per job (daily report=catch-up; every-min poll=skip stale) · **no-overlap** option · misfire policy.

## Retries
Backoff + jitter, capped → **DLQ + alert** (poison jobs) → redrive. Circuit breaker on bad downstream.

## Bottlenecks (ranked)
1. Due-scan → index / bucket / shard
2. Scheduler contention → SKIP LOCKED + partition
3. Execution → add stateless workers (queue buffers)
4. **Thundering herd @ :00 → JITTER** + per-tenant limits
5. Run history → append store + TTL

## Failure one-liners
- Worker dies → lease expiry → redeliver; dedup+fence = safe.
- **Scheduler down at fire time → durable next_run_at → catch-up on recovery** (per-job policy); leader-election standby.
- Two schedulers → atomic claim → one wins.
- Zombie worker → fencing token rejects stale write.
- Store/queue down → job stays CLAIMED/durable → delayed, not lost.
- Clock skew → NTP + store-based time; fencing is clock-independent.

## Patterns used
[queue+workers](../../01-patterns/queues-workers.md) · [idempotency](../../01-patterns/idempotency.md) ·
[sharding](../../01-patterns/sharding.md) · [rate limiting](../../01-patterns/rate-limiting.md) · distributed locks/fencing.

## SQL vs NoSQL
Relational schedule store (index + SKIP LOCKED + atomic claim = exactly-once primitives) → bucketed/sharded at
extreme scale. Run history → cheap append store + TTL.

## Summary line
> "Separate scheduling from execution. Scheduler finds due jobs via a next_run_at index (buckets at scale),
> atomically claims each so only one fires it, enqueues to a durable queue; stateless workers execute. Exactly-once
> *effect* via at-least-once + idempotency ledger + idempotent jobs, with leases + **fencing tokens** against zombies.
> Cron recomputes next fire with timezone/DST + catch-up-vs-skip. Partition + leader election prevent double-fire;
> jitter kills the top-of-hour herd; retries → DLQ."

← Back to **[README](README.md)**
