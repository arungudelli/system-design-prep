# Staff-Level Thinking

> Senior engineers produce a **correct** design. Staff engineers produce a design and then **attack it** —
> they know where it breaks, what it costs, what they'd simplify, and when they'd redesign it. The
> difference isn't more components; it's more **judgment**. This page is the judgment layer.

---

## The mindset shift

| Senior tends to… | Staff tends to… |
|---|---|
| Reach for the "best" tech | Reach for the **cheapest thing that meets the SLA**, and justify every upgrade |
| Design the happy path | Design the **failure paths** and blast-radius containment |
| Add Kafka/microservices/sharding | Ask *"what breaks if I **don't**?"* and defer until a requirement forces it |
| Answer "what" | Answer **"why this and not the simpler thing"** |
| Optimize a component | Reason about **boundaries** — what's coupled, what scales independently |
| State a decision | State the **trade-off**, argue both sides, then choose |

The tell: a staff candidate can be handed an *unfamiliar* problem and stay calm, because they're deriving,
not recalling.

---

## The "why not simpler?" reflex

For every component you add, out loud:

> **"Why do we need this? What problem does it solve that the simpler design cannot?"**

If you can't answer crisply, remove it. Complexity you can't justify is negative signal. The strongest
move in a system design interview is often *declining* to add something:

> "I could add Kafka here, but SQS meets our throughput and ordering needs and is far less to operate.
> I'd only reach for Kafka if we needed replay, high fan-out to many consumer groups, or >100k msg/s."

---

## The scaling interrogation (be ready for all of these)

For any design, the interviewer will probe. Rehearse crisp answers:

- Why **this** architecture? Why not something simpler?
- Where is the **bottleneck**? How would you detect it (which metric)?
- What happens at **10×** scale? At **100×**?
- Which component **fails first**?
- What happens when a **dependency becomes slow** (200 ms → 10 s)?
- What happens when a **region fails**?
- What if messages are **duplicated**? **lost**? processed **out of order**?
- What if a **worker crashes mid-way** through a job?
- What happens **during deployment**? during **DB failover**? during **cache failure**?
- How do you prevent **cascading failures**?
- How do you isolate **noisy tenants**?
- How do you handle **hot keys / hot partitions**?
- How do you ensure **backward compatibility**?
- What are the **operational** complexities?
- What is the **cheapest acceptable** architecture?
- **When would you redesign** this system?

You won't get to all of them — but if any one makes you freeze, that's your study target.

---

## The distributed-systems failure canon

These come up in nearly every design. Have a one-liner *and* a pattern for each:

| "What if…" | The pattern that answers it |
|---|---|
| A **retry duplicates** a write? | **Idempotency** — idempotency key + dedup table / unique constraint |
| The DB commits but the **event publish fails**? | **Transactional outbox** (write event in same txn, relay async) — avoids distributed transactions |
| **Two workers grab the same job**? | Lease / **fencing token**, `SELECT … FOR UPDATE SKIP LOCKED`, or visibility timeout |
| A **downstream slows down**? | Timeout + **circuit breaker** + bulkhead so it can't exhaust your threads |
| Producers **outpace** consumers? | **Backpressure** — bounded queue, load shedding, consumer autoscaling |
| A **hot key** melts one cache node? | Key splitting, local cache in front, request coalescing |
| A **hot shard** melts one DB node? | Better shard key, salting, split the range, consistent hashing |
| A **poison message** jams the queue? | Retry limit → **dead-letter queue (DLQ)** + alert |
| **Cache dies**? | Design to survive on DB (degraded), stampede protection on refill |
| **Region dies**? | Multi-AZ first; multi-region (active-passive/active-active) only if RTO/RPO demands it |

> Golden rule you should say out loud: **at-least-once delivery + idempotent consumers** is almost always
> the right answer, because true end-to-end exactly-once is extremely hard (the DB commit and the ack are
> two separate systems). Don't promise exactly-once; engineer idempotency instead.

---

## Architect-level boundary questions

After the design works, **zoom out** and answer these — this is pure staff signal:

1. What are the major **architectural boundaries** (services / modules)?
2. Which components should stay **tightly coupled** (change together, so keep together)?
3. Which deserve **independent scaling** (different load profiles)?
4. Which **failures need isolation** (bulkheads / separate pools)?
5. Which **data belongs to which service** (no shared DB across service boundaries)?
6. Where should **consistency boundaries** live (what's one transaction, what's eventual)?
7. Which paths must be **synchronous** (user is waiting)?
8. Which should be **asynchronous** (fire-and-forget, or user doesn't wait)?
9. Where are the **operational-complexity** risks (the thing that pages you at 3am)?
10. Which components should be **managed cloud services** vs custom-built?
11. Where could **vendor lock-in** bite, and is that trade worth it?
12. What should stay **deliberately simple**, forever?

A great closing line: *"The boundaries I care most about are X and Y — they scale independently and fail
independently. Everything else I'd keep together to reduce operational surface."*

---

## Coupling & consistency boundaries (the two hardest calls)

- **Coupling:** put things that *change together* and *fail together* on the same side of a boundary.
  Splitting them creates distributed transactions and chatty calls; a premature microservice is a
  distributed monolith with worse latency.
- **Consistency:** decide the **transaction boundary** deliberately. Inside a boundary → one ACID
  transaction, strong consistency. Across boundaries → events + eventual consistency + idempotency. Draw
  the line where the business genuinely needs atomicity (money, inventory, unique claims) and nowhere else.

---

## Cost as a design constraint

Staff engineers treat cost as a first-class NFR, not an afterthought:

- Name the **top cost driver** and which grows fastest with scale (usually CDN egress, DB replicas, Redis
  memory, or transcoding — depending on system).
- Ask: *"what's the cheapest architecture that still meets the SLA?"*
- Offer a cheaper alternative for the non-critical parts (e.g. cold storage tier, spot workers for batch).

> "Infinite budget isn't real. I'd spend on the p99 read path and the durability of payments, and keep
> everything else lean."

---

## When would you redesign? (the maturity question)

Have an answer ready. Redesign triggers are usually:

- A **dominant assumption changed** (read-heavy became write-heavy; single-region became global).
- The **shard key** no longer matches access patterns (resharding is painful — a sign the model was wrong).
- **Operational cost** (toil, incidents) exceeds the cost of rebuilding.
- A **new requirement** (real-time, compliance, multi-tenancy) breaks a core assumption.

Saying *"I'd redesign when X, and here's the migration path (dual-write / backfill / cutover)"* is elite signal.

---

## How your answer gets graded (the critique lens)

Interviewers (and your own self-review) evaluate:

1. Requirements clarification 2. Capacity estimation 3. API design 4. Data modeling 5. **Simplicity**
6. Scalability 7. Reliability 8. Consistency 9. **Failure handling** 10. Security 11. Observability
12. **Cost awareness** 13. **Trade-off reasoning** 14. Communication 15. **Staff-level judgment**

The **bold** ones are where senior candidates most often lose staff-level points. When you self-review a
mock, be brutal: *what was good, what was missing, what was needlessly complex, which assumption was
dangerous, what would an interviewer challenge, and what should I have said instead?* Don't praise a weak answer.

→ Next: **[Interview wording](00-framework/interview-wording.md)**
