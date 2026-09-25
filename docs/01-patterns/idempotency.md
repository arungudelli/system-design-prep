# Idempotency

> Idempotency is the quiet superpower of distributed systems: it's what makes **retries safe**. Networks time
> out, clients retry, queues redeliver, workers crash and restart — all of which cause the *same operation to
> happen more than once*. Idempotency ensures that doing an operation twice has the **same effect as doing it
> once**. It's the practical answer to "how do you get exactly-once?" — you don't; you make duplicates harmless.

**One-line mental model:** *at-least-once delivery + idempotent consumer = the **effect** of exactly-once.* That
sentence answers half the reliability follow-ups in a system design interview.

---

## Definition

An operation is **idempotent** if applying it multiple times produces the same result as applying it once.

- `GET /user/1` — idempotent (reading doesn't change state).
- `PUT balance = 100` — idempotent (setting to an absolute value; repeat = same).
- `POST charge $50` / `balance += 50` — **NOT** idempotent (each call changes state again → double charge).

The whole game is turning that last, dangerous kind of operation into something safe to repeat.

---

## Problem — why duplicates are unavoidable

You can't prevent duplicates; you can only make them harmless. They come from everywhere:

- **Client retries:** request times out, but the server *did* process it — the client retries → duplicate.
- **At-least-once queues:** the broker redelivers if a worker doesn't ack in time (see
  [queues & workers](queues-workers.md#acknowledgment--the-visibility-timeout-the-core-mechanic)).
- **Worker crash mid-job:** worker does the work, crashes before acking → message redelivered → reprocessed.
- **Load-balancer / proxy retries**, browser double-submits, mobile flaky networks, user double-clicks.
- **Network partitions:** caller can't tell "did it succeed?" from "did the response get lost?" → retries.

> The fundamental uncertainty: **a timeout does not tell you whether the operation happened.** So safe systems
> assume it might have, and make repeating it a no-op.

## When to use
- **Any state-changing operation that can be retried** — which is essentially all of them in a distributed system.
- **Money, inventory, jobs, notifications, message processing** — anywhere a duplicate has a visible/expensive effect.

## When you can relax
- **Naturally idempotent** operations (absolute-value writes, set membership, "mark as read") need no extra
  machinery — repeating them is already safe. Recognize these and don't over-engineer.
- **Reads** (GET) — inherently idempotent.

---

## Techniques (from simplest to most general)

### 1. Make the operation naturally idempotent (best when possible)
Prefer operations whose repeat is a no-op by construction:

- **Set, don't increment:** `SET status = 'PAID'` instead of a counter bump; `PUT balance = X` instead of `+= X`.
- **Upsert:** `INSERT … ON CONFLICT DO NOTHING` / `MERGE`.
- **Idempotent state transitions:** "move order to SHIPPED" is safe to repeat; "advance to next state" is not.

This is the cleanest option — no extra state to store. Use it wherever the domain allows.

### 2. Unique constraint (let the database enforce it)
Put a **unique index** on a natural or supplied key so a duplicate insert fails harmlessly.

```sql
-- A payment can be inserted only once per (order_id)
CREATE UNIQUE INDEX ux_payment_order ON payments(order_id);

INSERT INTO payments(order_id, amount) VALUES (123, 50)
ON CONFLICT (order_id) DO NOTHING;   -- second attempt is a no-op
```

- ✅ Simple, atomic, correct — the DB is the arbiter, no race.
- ❌ Requires a meaningful unique key; only covers "create-once" shapes.

### 3. Idempotency key (the general API pattern)
The client generates a unique key (UUID) per logical operation and sends it (e.g. `Idempotency-Key` header).
The server stores `key → result` and **returns the stored result on replay** instead of re-executing.

```text
POST /v1/charges
Idempotency-Key: 5f2c…-uuid
{ "amount": 5000, "currency": "usd", "source": "card_x" }
```

Server logic:
```text
on request(key, payload):
  row = store.get(key)
  if row exists:
      if row.status == DONE:   return row.saved_response      # replay → same result
      if row.status == IN_PROGRESS: return 409/425 (retry later)  # concurrent duplicate
  else:
      atomically insert (key, IN_PROGRESS)   # unique constraint guards the race
      result = do_the_work(payload)          # the side effect
      store.update(key, DONE, saved_response=result)
      return result
```

Key details interviewers probe:
- **Store the key *before* doing the work**, guarded by a unique constraint, so two concurrent duplicates can't
  both execute (one wins the insert, the other sees IN_PROGRESS).
- **Persist the response**, so a replay returns the *same* result (same charge id), not just "already done".
- **Scope + TTL:** keys are usually scoped per endpoint/account and expire after a window (e.g. 24h) — long
  enough to cover realistic retries, short enough to bound storage.
- **Bind the key to the payload** (hash it): if the same key arrives with a *different* body, reject it (`422`) —
  that's a client bug, not a legitimate retry.

This is exactly how **Stripe, PayPal, and most payment APIs** implement safe retries.

### 4. Processed-event store / dedup table (for message consumers)
For at-least-once queues, the consumer records each processed message ID and skips duplicates.

```text
on message(msg):
  if dedup_store.exists(msg.id):   return   # already processed → skip
  process(msg)                              # the effect
  dedup_store.add(msg.id, ttl)              # remember it
```

- **The ordering trap:** if you `process` then record, a crash in between → reprocess. If you record then
  process, a crash → the message is marked done but *wasn't* → lost. The robust fix is to make the **effect and
  the dedup-record atomic** — write the dedup row **in the same transaction** as the business effect (or use the
  operation's own unique constraint as the dedup). This is the message-side version of the outbox idea.
- **TTL the dedup store** to bound its size (keep IDs long enough that no realistic redelivery outlives them).
- **Dedup at scale:** a Redis `SET`/`SETNX` with TTL, a DB table, or a Bloom filter in front (accepting rare
  false-positives = rare dropped duplicates only if you can tolerate it).

---

## Worked examples (the ones interviewers ask for)

### Payments — the classic "don't double-charge"
A user clicks "Pay" and the response times out. The client retries. Without idempotency: **two charges.**
- Client sends an **idempotency key** with the charge. Server stores `key → charge_result`; the retry returns the
  *same* charge id, no second charge.
- Belt-and-suspenders: a **unique constraint** on `(order_id)` in the payments table so even a bug can't
  double-insert.
- **Never** model a charge as `balance += amount` behind a retryable call without one of these guards.

### Background jobs — "two workers grabbed the same job"
A job is redelivered (visibility timeout expired while a slow worker was still running) → two workers run it.
- Guard the **effect** with a unique key (e.g. `job_id` in a `completed_jobs` table, or a conditional update
  `UPDATE jobs SET status='DONE' WHERE id=? AND status='RUNNING'` — only one wins).
- Or use a **lease with a fencing token** so a stale worker's write is rejected (see
  distributed locks *(coming)*).

### Notifications — "don't send the same email twice"
Retry/redelivery could send a duplicate push/email.
- Dedup on a **notification id** (e.g. `hash(user, event, template, day)`), recorded before/with the send.
- Note: external send is a side effect you can't roll back — so record intent, and prefer providers that accept
  a client-side **idempotency key** for the send too. Some duplicate risk always remains → make dedup best-effort
  tight, and accept at-least-once for non-critical channels.

### API retries — safe by contract
- Make **PUT/DELETE idempotent** by HTTP semantics; make **POST** idempotent with an `Idempotency-Key`.
- Return the stored response on replay so clients can safely retry any 5xx/timeout.

### Message processing — the queue consumer
See technique #4: at-least-once + dedup/unique-constraint → the effect of exactly-once. This is *the* reason
[queues & workers](queues-workers.md#idempotency-the-essential-partner--retries-make-this-mandatory)
insists consumers be idempotent.

---

## The concurrency race (don't miss this)

Two duplicates can arrive **at the same time** (client double-fires, or two workers get redeliveries). A naive
"check-then-act" (`if not exists: do it`) has a race — both checks pass before either writes. Fixes:

- **Unique constraint / conditional insert** — let the DB serialize it; one insert wins, the other conflicts.
- **`SETNX`/atomic compare-and-set** in Redis for a lock/marker.
- **`SELECT … FOR UPDATE` / `INSERT … ON CONFLICT`** — atomic in one statement.

> Idempotency isn't just "remember what we did" — it's "remember it **atomically with doing it**, and handle two
> attempts racing." State that and you've shown staff-level depth.

---

## Trade-offs

- **Natural idempotency vs added machinery:** if you can design the operation to be naturally idempotent
  (set/upsert), do that — zero extra state. Reach for keys/dedup tables only when you can't.
- **Storage + TTL:** idempotency keys and dedup records cost storage; TTL bounds it but too-short a TTL lets a
  late retry slip through. Size the window to your realistic retry horizon.
- **Exactly-once effort vs at-least-once + idempotency:** chasing true exactly-once delivery is a trap; investing
  in idempotent consumers is cheaper and actually works.
- **Strictness vs cost:** a Bloom filter dedup is cheap but can false-positive; a DB unique constraint is exact
  but adds a write. Choose per how costly a duplicate is (a double email vs a double charge).

---

## Interview wording

> "A timeout doesn't tell the client whether the operation succeeded, so it must be safe to retry — I'll make the
> operation idempotent."
> "For the charge API I'd require an idempotency key: we store key → result and return the same result on retry,
> so a network retry never double-charges."
> "Where I can, I'll design the operation to be naturally idempotent — set an absolute value or upsert — so
> repeating it is inherently a no-op."
> "The queue is at-least-once, so consumers must be idempotent — I dedup on the message id, recorded in the same
> transaction as the effect, which gives us exactly-once *effect* without exactly-once delivery."
> "I'd guard the create with a unique constraint so even a concurrent duplicate can't insert twice — the database
> arbitrates the race, not my application code."
> "For the job, a conditional update `WHERE status='RUNNING'` ensures only one of two workers commits the result."

---

## Where it appears (across systems)

- **Payments** — idempotency keys + unique constraints; the canonical example.
- **Job scheduler** — safe redelivery; conditional updates / leases + fencing tokens.
- **Notification system** — dedup on notification id to avoid double-sends.
- **Any queue consumer** — the mandatory partner to at-least-once
  ([queues & workers](queues-workers.md)).
- **URL shortener** — `Idempotency-Key` on `POST /urls` so a retried create doesn't mint two codes
  ([API & data model](../02-systems/url-shortener/api-data-model.md#idempotency-why-the-header-matters)).
- **Transactional outbox** — pairs with idempotent consumers to make event delivery reliable
  ([outbox](queues-workers.md#the-transactional-outbox-dont-lose-this-one--its-a-top-follow-up)).

---

## Quick checklist

- [ ] Can the operation be made **naturally idempotent** (set/upsert)? Do that first.
- [ ] If not, is there a **unique key** to enforce with a DB constraint?
- [ ] For APIs: accept an **Idempotency-Key**, store `key → response`, replay the stored result.
- [ ] For consumers: **dedup on message id**, recorded **atomically with the effect**.
- [ ] Handle the **concurrent-duplicate race** (unique constraint / SETNX / conditional update), not just
      sequential replays.
- [ ] **TTL** the keys/dedup store to bound storage — but cover the realistic retry window.
- [ ] Bind the idempotency key to the **payload** so a reused key with a different body is rejected.

← Back to **[patterns](../index.md)** · Related: [queues & workers](queues-workers.md) · distributed locks *(coming)*
