# Queues & Workers (Async Processing)

> A queue is how you stop making the user wait for work that doesn't need to finish before you reply. It's also
> how you absorb bursts, decouple services, and retry failures safely. The interview signal isn't "I'd add a
> queue" — it's knowing **delivery semantics, retries, visibility timeouts, DLQs, backpressure**, and being able
> to answer *"when is SQS enough, and what requirement justifies Kafka?"* without reflexively saying "Kafka."

**One-line mental model:** a queue turns a **synchronous, coupled, fragile** call into an **asynchronous,
decoupled, retryable** one — trading immediate consistency and simplicity for throughput, resilience, and buffering.

---

## Problem

Some work is slow, bursty, failure-prone, or simply not needed on the user's critical path (sending an email,
transcoding a video, updating a search index, charging a card, fanning out a notification). Doing it
synchronously makes the user wait, couples the caller's availability to the worker's, and loses work if the
worker is briefly down.

## When to use
- Work can be done **after** the response (the user doesn't need the result now).
- **Smooth bursts:** producers spike faster than consumers can handle → the queue is a shock absorber.
- **Decouple** producer and consumer (deploy/scale/fail independently).
- **Retry** flaky work safely without blocking the caller.
- **Fan-out:** one event, many independent consumers.

## When NOT to use
- The caller **needs the result synchronously** (a read the user is waiting on) → a queue just adds latency.
- The work is **trivial and reliable** → the queue is operational overhead for no benefit.
- You need a **strict, immediate, transactional** guarantee across the call → async makes consistency eventual;
  design for that or don't go async.
- You reach for Kafka when a simple task queue (SQS) would do → complexity with no matching requirement.

---

## The anatomy

```text
 Producer(s) ──put──► [ Broker / Queue ] ──deliver──► Consumer(s) / Workers
                          |  buffers, orders(ish),        |  process, then ACK
                          |  retries, DLQ                 |
                          └───────── ack / nack ──────────┘
                                                          │ on repeated failure
                                                          ▼
                                                    Dead-Letter Queue
```

- **Producer** — enqueues a message/event. Should be fast and fire-and-forget-ish.
- **Broker / queue** — durably holds messages, handles delivery, retries, ordering, DLQ. (SQS, RabbitMQ, Kafka.)
- **Consumer / worker** — pulls (or is pushed) a message, does the work, then **acknowledges**. Scaled horizontally.
- **Acknowledgment (ack)** — "I finished; delete it." Until ack, the broker keeps/redelivers the message.

---

## Acknowledgment & the visibility timeout (the core mechanic)

The central trick that makes queues reliable: a message isn't deleted when *delivered* — only when *acked*.

- **Visibility timeout / lease** (SQS/RabbitMQ style): when a worker receives a message, it becomes **invisible**
  to other workers for a timeout. The worker must **ack before the timeout** or the message becomes visible again
  and is **redelivered** (assumed the worker died). This is what makes "worker crashes mid-job" safe.
- **Consumer offsets** (Kafka style): each consumer group tracks an **offset** (position in the log). "Ack" =
  committing the offset. Crash before commit → reprocess from the last committed offset.

**Consequences you must design for:**
- **Set the visibility timeout > worst-case processing time**, or a slow job gets redelivered and runs twice.
- Long jobs → **extend the lease** (heartbeat) while working, or the message reappears mid-flight.
- Because redelivery happens, **consumers must be idempotent** (see below).

---

## Delivery semantics (know all three cold)

| Semantic | Guarantee | Cost | Reality |
|---|---|---|---|
| **At-most-once** | Never processed twice; **may be lost** | Simplest, no retries | Acceptable only when loss is OK (some metrics/telemetry) |
| **At-least-once** | Never lost; **may be processed >1×** | Retries + dedup | **The practical default** |
| **Exactly-once** | Processed exactly one time | Very hard end-to-end | Mostly a myth across systems; approximated |

### Why end-to-end exactly-once is (essentially) impossible
The classic gap: a worker finishes the work and then must **ack**. If it crashes *between* doing the work and
acking, the message is redelivered → the work happens again. The DB commit and the queue ack are **two separate
systems**; you can't atomically do both. So you can't truly guarantee exactly-once across process + ack.

**What to say (the golden rule):**
> *"I'd use **at-least-once delivery with idempotent consumers**. True end-to-end exactly-once is impractical
> because the work and the ack live in different systems; making the consumer idempotent gives the same
> *effect* as exactly-once, which is what actually matters."*

Kafka's "exactly-once semantics" (EOS) is real but **only within Kafka** (transactions across topics/offsets);
the moment you touch an external DB or API, you're back to at-least-once + idempotency.

---

## Idempotency (the essential partner — retries make this mandatory)

At-least-once means the same message *will* be processed more than once eventually. The consumer must make
reprocessing a **no-op**:

- **Idempotency key / dedup table:** record a processed message ID; skip if already seen.
- **Unique constraint / upsert:** `INSERT … ON CONFLICT DO NOTHING`, or make the operation naturally idempotent
  (set-to-X instead of increment-by-1 where possible).
- **Conditional writes:** apply only if state hasn't already advanced.

Full treatment in [idempotency](idempotency.md) *(coming)*. The one-liner: **at-least-once + idempotent
consumer = the effect of exactly-once.**

---

## Retries, backoff, and the DLQ

- **Retry** transient failures (a downstream blip) — but with **exponential backoff + jitter** so you don't
  hammer a struggling dependency or create a synchronized retry storm.
- **Cap retries.** Infinite retries on a message that will *never* succeed jams the queue.
- **Poison message** — a message that fails every time (malformed, references deleted data, triggers a bug). It
  will retry forever and block progress if you let it.
- **Dead-Letter Queue (DLQ)** — after N failed attempts, move the message to a separate queue for inspection
  instead of retrying forever. **Alert on DLQ depth > 0** — it means something needs a human.
- **Redrive:** after fixing the bug/data, replay DLQ messages back into the main queue.

```text
message fails → retry w/ backoff → fails again … → after N attempts → DLQ (+ alert)
```

> **Interview line:** *"Transient failures retry with exponential backoff and jitter; after a few attempts a
> message goes to a DLQ and we alert, so a poison message can't block the queue. Once fixed, we redrive the DLQ."*

---

## Backpressure (protecting yourself when producers outrun consumers)

If producers persistently produce faster than consumers consume, the queue grows unboundedly → memory blows up,
latency skyrockets, the system falls over. Backpressure = mechanisms to keep the system stable under overload.

- **Bounded queues** — cap the queue; when full, apply a policy (reject, block, or drop-oldest).
- **Consumer autoscaling** — scale workers on **queue depth / consumer lag / oldest-message age** (not CPU).
- **Load shedding** — reject or drop low-priority work early (return `429`), so critical work still flows.
- **Throttling / rate limiting** at the producer (see [rate limiting](rate-limiting.md) *(coming)*).
- **Buffering** — a bounded buffer smooths short spikes; it does **not** fix a sustained imbalance (only more
  consumers or less load does).

**Key metric:** **queue depth / consumer lag rising steadily = consumers can't keep up.** Alert on it and on
**oldest-message age** (SLA for how stale processing may get). See observability *(coming)*.

> **Interview line:** *"I'd autoscale consumers on queue depth and alert on the oldest-message age. If we still
> can't keep up, I shed low-priority messages so the critical path stays healthy — a bounded queue plus load
> shedding prevents an unbounded backlog from taking the system down."*

---

## Ordering (a common gotcha)

- Most queues give **best-effort or per-partition ordering**, not global ordering.
- **Kafka:** ordered **within a partition**; choose the partition key so messages that must be ordered share one
  (e.g. key by `user_id` → all of a user's events are ordered). Global order across partitions is not guaranteed.
- **SQS standard:** no ordering guarantee. **SQS FIFO:** ordering + dedup within a *message group*, at lower throughput.
- **RabbitMQ:** ordered per queue, but redelivery/multiple consumers can reorder.
- **If you need ordering,** partition by the entity that needs it — and remember ordering + parallelism trade off
  (strict order limits how much you can parallelize).

---

## The transactional outbox (don't lose this one — it's a top follow-up)

**Problem:** you need to update the DB **and** publish an event. If you write the DB then publish and crash in
between, the event is lost (state changed, no one told). If you publish then write and crash, you emitted an
event for a change that didn't happen. Two systems, no atomic write → the **dual-write problem**.

**Solution — transactional outbox:**
1. In the **same DB transaction** as your state change, insert the event into an `outbox` table.
2. A **relay** (poller or CDC/change-data-capture) reads the outbox and publishes to the broker, marking rows sent.
3. At-least-once publish + idempotent consumers handle any duplicate from a relay retry.

```text
BEGIN TX
  update business row
  insert into outbox(event)     ← same transaction, atomic
COMMIT
        │
   relay/CDC polls outbox ──► publish to broker ──► consumers (idempotent)
```

> *"To avoid the write-then-publish gap I'd use a transactional outbox — the event is written in the same
> transaction as the state change, then relayed to the broker — rather than a distributed transaction."*

---

## SQS vs RabbitMQ vs Kafka (the decision — never just say "Kafka")

| | **SQS** | **RabbitMQ** | **Kafka** |
|---|---|---|---|
| Model | Managed **task queue** | **Message broker** (exchanges/routing) | **Distributed log** (durable, replayable) |
| Consume | Pull, visibility timeout, message deleted on ack | Push/pull, ack, flexible routing | Consumers read by **offset**; messages **retained** (not deleted on read) |
| Ordering | Standard: none; FIFO: per group | Per queue | Per **partition** |
| Replay / history | ❌ (gone after ack) | ❌ (gone after ack) | ✅ **replay** from any offset; multiple independent consumer groups |
| Throughput | Very high, elastic, hands-off | High | **Very high** (millions/s), horizontal |
| Fan-out to many consumers | Limited (use SNS+SQS) | Via exchanges | ✅ native (many consumer groups read the same log) |
| Ops burden | **Lowest** (fully managed) | Medium (run a cluster) | **Highest** (partitions, brokers, ZK/KRaft, retention) |
| Best for | Simple decoupling, task queues, retries, DLQ | Complex routing, per-message ack, RPC-ish | Event streaming, replay, high fan-out, log of record, stream processing |

**When is SQS enough?** Most background-job / decoupling use cases: send email, process upload, retry a webhook.
Managed, cheap, DLQ built in, scales itself. **Reach for it by default.**

**What requirement justifies Kafka?** You need one or more of: **event replay** (reprocess history / new consumer
reads from the start), **high fan-out** (many independent consumer groups on the same stream), **very high sustained
throughput** (100k+ msg/s), an **ordered durable log as the source of truth**, or **stream processing**. If none
of those apply, Kafka is operational cost with no matching benefit.

**When RabbitMQ?** Rich **routing** (topic/fanout/direct exchanges), per-message priorities, request/reply
patterns, and you want more control than SQS without Kafka's log semantics.

> **Interview line:** *"I'd start with SQS — it's managed, has retries and a DLQ, and meets our throughput. I'd
> only move to Kafka if we needed event replay, high fan-out to many consumer groups, or streaming-scale
> throughput. Saying 'Kafka is more scalable' isn't a reason; the reason is a specific requirement it uniquely serves."*

---

## Architecture (typical async job flow)

```text
Client → API (fast, synchronous) ──enqueue──► Queue ──► [ Worker pool (autoscaled) ]
   │                                            │              │ idempotent processing
   └── 202 Accepted + jobId (poll for status)   │              │ ack on success
                                                 │              ▼ on repeated failure
                                                 │           DLQ (+ alert, redrive)
                                            (autoscale workers on queue depth)
```

- API returns **`202 Accepted` + a job id** immediately; client polls a status endpoint or gets a callback/webhook.
- Workers scale on **queue depth**, process idempotently, ack on success, DLQ on repeated failure.

---

## Trade-offs (argue both sides)

- **Sync vs async:** async gives responsiveness + resilience + burst absorption, at the cost of **eventual
  consistency**, more moving parts, and harder debugging (where did my message go?). Only go async when the work
  truly doesn't need to block the response.
- **At-least-once + idempotency vs chasing exactly-once:** the former is simpler and achievable; the latter is a
  trap. Choose idempotency.
- **Push vs pull:** pull (SQS/Kafka) lets consumers control their own rate (natural backpressure); push
  (RabbitMQ) is lower-latency but can overwhelm a slow consumer. See push vs pull *(coming)*.
- **Ordering vs throughput:** strict ordering limits parallelism; relax ordering where you can.
- **Managed (SQS) vs self-run (Kafka/RabbitMQ):** less ops vs more control/features. Default to managed.

---

## Where it appears (across systems)

- **Web crawler** — the **URL frontier** is a queue of URLs to fetch; workers = fetchers; politeness + dedup around it.
- **Notification system** — enqueue notifications; workers fan out to email/SMS/push; retries + DLQ per channel.
- **Job scheduler** — due jobs enqueued to workers; visibility timeout / lease so a crashed worker's job is redelivered.
- **YouTube / file storage** — upload triggers async **transcoding** jobs.
- **Payments** — async capture/settlement; **outbox** for reliable event emission; idempotency everywhere.
- **URL shortener** — click **analytics events** to a queue, off the redirect critical path.

---

## Quick checklist

- [ ] Does this work **need** to be sync? If not, enqueue and return `202`.
- [ ] **Delivery:** at-least-once + **idempotent consumer** (dedup key / upsert)?
- [ ] **Visibility timeout > processing time**; heartbeat/extend for long jobs?
- [ ] **Retries** with backoff + jitter, **capped**, then **DLQ + alert**?
- [ ] **Backpressure:** autoscale on queue depth; bounded queue; load-shed low priority?
- [ ] **Ordering** needed? Partition by the entity that requires it.
- [ ] Publishing an event with a DB write? Use the **transactional outbox**.
- [ ] **Broker choice** justified by a requirement (SQS default; Kafka only for replay/fan-out/throughput)?
- [ ] **Observe:** queue depth, consumer lag, oldest-message age, DLQ count.

← Back to **[patterns](../index.md)** · Related: [idempotency](idempotency.md) *(coming)* · [caching](caching.md) · [sharding](sharding.md)
