# Distributed Job Scheduler — Capacity Estimation

> See the method in **[Capacity estimation](../../00-framework/capacity-estimation.md)**. The numbers that shape a
> scheduler: **jobs firing/sec** (→ worker count + queue throughput), the **due-scan rate** (→ how you index "what's
> due"), and **stored jobs** (→ schedule store size). Note the asymmetry: you may *store* millions of jobs but only
> *fire* a small fraction per second.

---

## Assumptions (state them)

- **100M scheduled jobs** live in the system (recurring + pending one-off).
- Peak **fires ≈ 10,000 jobs/sec** (state your own; this is a healthy mid-scale number).
- Average job execution time: **~200 ms** (mix of quick webhooks + some longer tasks).
- Scheduler polls the "due" index every **~1 second** (tick).

---

## Fire rate → workers + queue throughput

```text
peak fires ≈ 10,000 jobs/sec
```

**Worker math (execution is often I/O-bound):**
```text
if a job takes ~200 ms, one worker slot does ~5 jobs/sec
to sustain 10,000/sec → ~2,000 concurrent slots
with async I/O (many slots per machine), that's a modest fleet; scale by adding workers
```

→ **So what?** Execution throughput scales **horizontally** by adding stateless workers pulling from a queue. The
queue absorbs bursts (10k/s fires don't all need to execute instantly). This is a
[queue + workers](../../01-patterns/queues-workers.md) problem at its core.

---

## The due-scan (the interesting number)

The scheduler must repeatedly answer: **"which jobs are due now?"** Naively scanning 100M jobs every second is
catastrophic.

```text
100M jobs, tick every 1s → scanning all 100M/s = impossible
```

→ **So what?** You need a **time-ordered index** so the query is `WHERE next_run_at <= now` and touches only the
**due** rows (a few thousand), not all 100M. Options:
- A **DB index on `next_run_at`** → each tick reads only the small due slice.
- **Time bucketing** — partition jobs into per-minute (or per-second) buckets; each tick reads only the current
  bucket(s). Great for sharding and for spreading load.

Either way, the cost per tick is proportional to **jobs due**, not **jobs stored** — that's the whole trick. See
[Deep Dives](deep-dives.md#1-finding-what-is-due-efficiently).

---

## Storage

```text
per job: id, owner, schedule/cron, next_run_at, payload, status, retries, ... ≈ ~1 KB
100M jobs × 1 KB ≈ 100 GB   (schedule store)
+ run history: fires/day × record size (can dwarf the schedule store if kept long)
```

```text
run history: 10k/s × 86,400 s ≈ ~860M runs/day
at ~500 bytes/run ≈ ~430 GB/day if we keep full history
```

→ **So what?** The **schedule store (~100 GB)** fits comfortably (and shards cleanly). **Run history is the size
driver** — it grows fast, so apply **retention/TTL** (keep recent runs hot, archive/expire old) or you'll drown in
history. Store history in a cheap append-only store (or object storage), not the hot schedule DB.

---

## Summary — what the numbers told us

| Number | Value | Design consequence |
|---|---|---|
| Fires/sec | ~10k peak | Queue + horizontally-scaled stateless workers |
| Stored jobs | 100M (~100 GB) | Fits/shards fine; **not** the bottleneck |
| Due-scan | can't scan all 100M/tick | **Time-ordered index / bucketing** — cost ∝ jobs *due* |
| Run history | ~430 GB/day | **Retention/TTL** + cheap append store; the real size driver |

The profile: **coordination + reliability, moderate throughput** — the hard part is *correctness under crashes and
time*, not raw QPS. Design accordingly.

→ Next: **[API & Data Model](api-data-model.md)**
