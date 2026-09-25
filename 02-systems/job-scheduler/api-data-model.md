# Distributed Job Scheduler — API & Data Model

## APIs

```text
POST /v1/jobs                      # schedule a job
Idempotency-Key: <uuid>            # so a retried create doesn't schedule twice
{
  "type": "webhook",              # or a known task type
  "target": "https://…/hook",
  "payload": { ... },
  "schedule": { "runAt": "2026-09-26T02:00:00Z" }   # one-off
      // OR: { "cron": "0 2 * * *", "timezone": "America/New_York" }  # recurring
  "maxRetries": 5
}
→ 201 { "jobId": "…", "nextRunAt": "2026-09-26T02:00:00Z", "status": "SCHEDULED" }

GET    /v1/jobs/{id}               → job + status + nextRunAt
GET    /v1/jobs/{id}/runs          → execution history (paginated)
PATCH  /v1/jobs/{id}               → update schedule/payload
DELETE /v1/jobs/{id}               → cancel (stop future runs)
```

- **Idempotency-Key on create** so a client retry doesn't create two identical schedules
  (see [idempotency](01-patterns/idempotency.md)).
- **Async by nature** — creating a job returns immediately; execution happens later. History is a paginated read.
- **Cron + timezone** for recurring — timezone matters (DST), so store it explicitly.

---

## Data model

### Access patterns first (this decides everything)

1. **"Which jobs are due now?"** — the hot query, every tick. `WHERE next_run_at <= now AND status = SCHEDULED`.
   → needs a **time-ordered index** (or bucketing).
2. `get/update/cancel job by id` — point lookup.
3. `list a user's jobs` / `job run history` — by owner / by job.
4. **Claim a due job atomically** — conditional update (the exactly-once guard).

Pattern #1 (the due-scan) dominates and dictates the primary index.

### `jobs` (the schedule store)

```text
jobs
------------------------------------------------------------------
job_id          PK
owner_id / tenant_id
type            (webhook | task-type)
target, payload
schedule_kind   (ONCE | CRON)
cron_expr, timezone        (null for one-off)
next_run_at     TIMESTAMP  ◄── INDEXED (the due-scan key)
status          (SCHEDULED | CLAIMED | RUNNING | SUCCEEDED | FAILED | CANCELLED)
lease_owner     (which worker/scheduler currently holds it)
lease_expires_at
fence_token     BIGINT     (monotonic; bumped each claim — see fencing)
attempts, max_retries
created_at, updated_at
------------------------------------------------------------------
INDEX (next_run_at, status)         -- the due-scan
INDEX (owner_id)                    -- list a user's jobs
```

- **`next_run_at` is the linchpin.** Indexed so the due-scan reads only due rows. For recurring jobs, `next_run_at`
  is **recomputed and updated** after each fire.
- **`status` + `lease_*` + `fence_token`** implement the claim/lease/fencing that gives exactly-once *effect*.
- **Shard by `job_id`** (or by time bucket) — see below.

### `job_runs` (execution history)

```text
job_runs
------------------------------------------------------------------
run_id          PK
job_id          (FK, indexed)
scheduled_for   TIMESTAMP
started_at, finished_at
status          (SUCCEEDED | FAILED | TIMED_OUT)
attempt
worker_id, fence_token
result / error
------------------------------------------------------------------
```

- **Append-only + large** — this is the size driver (see [capacity](02-systems/job-scheduler/capacity.md#storage)).
  Apply **TTL/retention**; consider a cheap append store / object storage for old history.
- Serves "job history" reads and debugging; also the **idempotency ledger** ("did we already run this fire?").

---

## The "due index" — two ways to model it

The single most important modeling choice: how to find due jobs fast.

### A) Indexed column (simple, DB-native)
`INDEX (next_run_at)` + `SELECT ... WHERE next_run_at <= now ... FOR UPDATE SKIP LOCKED LIMIT N`.
- ✅ Simple; the DB does the work; `SKIP LOCKED` lets many schedulers grab disjoint due rows without blocking.
- ❌ At extreme fire rates the index hot-spots on "now"; mitigate with sharding/bucketing.

### B) Time bucketing (scales + shards naturally)
Put jobs into buckets keyed by fire time (e.g. per-minute: `bucket = floor(next_run_at / 60s)`). Each tick reads
**only the current bucket(s)**.
- ✅ Each tick touches a tiny, bounded set; buckets shard cleanly across scheduler nodes; great for millions of jobs.
- ✅ Maps well onto a KV store (e.g. a sorted set per bucket in Redis, or a partitioned table).
- ❌ Slightly more machinery; must handle jobs that span bucket boundaries and catch-up of old buckets.

> **Interview line:** *"I'd index jobs by `next_run_at` and pull due rows with `SELECT … FOR UPDATE SKIP LOCKED` so
> multiple schedulers grab disjoint work without contention. At larger scale I'd bucket jobs by fire-minute so each
> tick reads only the current bucket and buckets shard across scheduler nodes."*

---

## SQL or NoSQL?

- **Schedule store → relational (Postgres/MySQL) is a great fit** at moderate scale: you get the `next_run_at`
  index, **atomic conditional claims** (`UPDATE … WHERE status='SCHEDULED'`), `SELECT … FOR UPDATE SKIP LOCKED`, and
  transactions to update status + compute next run atomically. These primitives *are* the exactly-once machinery.
- **At very large scale → a partitioned/bucketed store** (a sharded DB, or Redis sorted sets per time bucket +
  a durable backing store). You trade some transactional convenience for horizontal scale.
- **Run history → cheap append store / columnar / object storage** with TTL (it's write-heavy and huge).

> **Interview line:** *"I'd start with a relational schedule store — its indexes, `SKIP LOCKED`, and atomic
> conditional updates give me the claim-and-exactly-once-effect primitives for free. I'd move to time-bucketed
> sharding only when a single DB can't hold the due-scan load. Run history goes to a cheap append store with TTL."*

→ Next: **[Architecture](02-systems/job-scheduler/architecture.md)**
