# The 20-Step System Design Framework

> The point of a framework isn't to fill 45 minutes with ritual. It's so that when you get an
> **unfamiliar** problem, you never freeze — you always know the next question to ask. Interviewers
> aren't grading the diagram; they're grading whether you can *derive* the diagram.

Use this as a **checklist you glance at**, not a script you recite. In a real interview you'll spend
most time on steps 1–5 (framing) and 11–15 (scaling + failure) — that's where staff-level signal lives.

---

## The one-line version

**Requirements → scale → data → simplest design → find the bottleneck → scale that one thing → repeat → break it on purpose.**

Everything below is that sentence, expanded.

---

## Phase A — Frame the problem (steps 1–5)

> ~25% of your time. If you skip this, every later decision is a guess. **Do not draw boxes yet.**

### 1. Clarify requirements
Restate the problem in your own words and confirm scope. The prompt is *deliberately* vague — that's the
test. A senior engineer starts coding; a staff engineer first asks *"what exactly are we building, and
what are we explicitly **not** building?"*

Say: *"Before designing, I'd like to clarify requirements and agree on scope."*

### 2. Functional requirements (what it does)
The verbs. *"Users can shorten a URL and be redirected."* Keep the **must-have** list to 2–4 items.
Everything else is **nice-to-have** — name it, then explicitly defer it: *"Analytics and custom aliases
are valuable but I'll defer them to keep scope tight; I can add them at the end if we have time."*

Deferring is a skill. Scope explosion is the #1 way candidates run out of time.

### 3. Non-functional requirements (how well it does it)
The adjectives — but **quantified**. This is where you separate yourself:

- **Latency** — "p99 redirect < 50 ms", not "fast"
- **Availability** — "99.9%" (≈ 43 min/month down) vs "99.99%" (≈ 4 min/month) — the extra nine changes the architecture
- **Consistency** — where is stale data OK, where is it not?
- **Durability** — "we must never lose a committed payment"
- **Scale / throughput** — read-heavy? write-heavy? (drives everything)

Pick the **2–3 NFRs that dominate** this system and say so. A URL shortener is read-heavy + latency-sensitive.
A payment system is durability + consistency. Naming the dominant NFR tells the interviewer you know what matters.

### 4. Define assumptions
Make the ambiguous concrete, out loud, and get a nod. *"I'll assume 100M new URLs/month and a 100:1
read:write ratio unless you'd like different numbers."* This lets you move fast **and** shows the
interviewer your reasoning is parameterized — if they change a number, you know what changes downstream.

### 5. Estimate scale
Order-of-magnitude only. See **[Capacity estimation](00-framework/capacity-estimation.md)** for the method.
The output you actually need:

- **Peak RPS** (reads and writes separately)
- **Storage/year**
- **Bandwidth** (if media is involved)

And for each number: *"This matters because…"* — e.g. *"40k read RPS means a single Postgres won't
serve reads directly; that's why I'll add a cache."* A number you don't use is theatre. Only compute
numbers that **change a decision**.

---

## Phase B — Model the domain (steps 6–8)

### 6. Core entities
The nouns. `User`, `URL`, `Job`, `Message`, `Payment`, `Notification`. Keep it to the handful that matter.
This quietly seeds your data model and API.

### 7. Define APIs
The contract between client and system. A few realistic endpoints:

```text
POST /v1/urls            { "longUrl": "..." }  ->  201 { "shortCode": "aB3x9" }
GET  /{shortCode}                              ->  302 Location: <longUrl>
```

Cover, where relevant: **pagination** (cursor > offset at scale), **idempotency keys** (writes that
retry — payments, job submits), **async APIs** (return `202 Accepted` + a status endpoint for long work),
and **error behavior**. Mention **REST vs gRPC**: REST for public/edge, gRPC for internal service-to-service
(binary, streaming, typed contracts). Don't over-design the API — 3–5 endpoints is plenty.

### 8. Data model
**Ask "what are our dominant access patterns?" *before* choosing a database.** The access pattern picks the
store; popularity doesn't. Then show a basic schema and call out:

- **primary key**, **indexes** (which queries they serve)
- **partition / shard key** (the single most important choice at scale — get it wrong and you get hot shards)
- **secondary indexes** and their cost

SQL vs NoSQL is a *consequence* of access patterns + consistency needs, not a coin flip.
See [SQL vs NoSQL](03-tradeoffs/sql-vs-nosql.md) *(coming)*.

---

## Phase C — Design, simplest first (steps 9–10)

### 9. Draw the simplest architecture that meets the requirements
Start boring. Resist the urge to impress.

```text
        Client
          |
      Load Balancer
          |
   [ Stateless API servers ]   (N of them)
          |
       Database
```

**Explain every box, and don't add a box you can't justify.** If you draw a cache, you must be able to say
which read it serves and what the hit ratio buys you. An unjustified component is a red flag, not a plus.

> "I'll start with the simplest architecture that satisfies these requirements, then evolve it as we find bottlenecks."

### 10. Walk the request / data flow
Trace the actual paths step by step:

- **Write flow** — what happens end-to-end when data is created
- **Read flow** — the hot path; optimize this one
- **Background flow** — anything async (email, transcode, indexing)
- **Failure flow** — what a request does when a dependency is down

Explicitly label **synchronous vs asynchronous**: *"The redirect is synchronous and must be <50 ms. Recording
the click analytic is not on the critical path, so I'll fire it asynchronously."*

---

## Phase D — Scale it (steps 11–12)

> This is the heart of the interview. Don't scale everything — **find the first thing that breaks, scale
> only that, then ask what breaks next.** Scaling is iterative, not a big-bang redesign.

### 11. Identify bottlenecks
Rank them. For **each** bottleneck answer four things:

1. **Why** does it become the bottleneck?
2. **How do we detect it?** (which metric)
3. **What's the signal?** (queue depth rising, p99 climbing, connection pool exhausted)
4. **How do we scale it?**

Usual suspects: CPU, memory, **DB connections**, **DB writes**, cache hot keys, queue depth, network
bandwidth, external-API limits. The database is the bottleneck in the majority of designs — expect it.

### 12. Scale each bottleneck
The standard toolkit, cheapest-first:

- **Stateless services** → horizontal scale behind the LB + autoscaling. (Push session state out to Redis so any server can handle any request.)
- **Reads** → cache (see [caching](01-patterns/caching.md) *(coming)*) → read replicas
- **Writes** → batching, write-behind, then **partition / shard** (last resort — it introduces cross-shard queries, hot shards, rebalancing)
- **Slow synchronous work** → move to a **queue + workers** (async)

Say the discipline out loud: *"Before sharding I'd first exhaust indexing, caching, and read replicas."*
Also note when **vertical scaling is simpler** — sometimes a bigger box buys you a year and costs less than
the operational complexity of sharding.

---

## Phase E — Prove it's production-grade (steps 13–20)

> Seniors design the happy path. Staff engineers design the **unhappy** path. Spend real time here.

### 13. Reliability
Timeouts, retries **with exponential backoff + jitter**, idempotency so retries are safe, circuit breakers,
bulkheads, graceful degradation. "Retry" without "idempotent consumer" is a bug you just designed in.

### 14. Consistency
State it **per data type**: what needs strong consistency (a payment balance, a username claim) vs what
can be eventually consistent (a follower count, a click tally). Mention **read-your-writes** and
**monotonic reads** where the UX cares. Bring up **CAP** only when it actually forces a choice — not as trivia.

### 15. Failure scenarios
Walk the blast radius explicitly: API server dies, worker dies mid-job, DB primary fails over, cache dies,
queue is unavailable, downstream slows from 200 ms → 10 s, an AZ fails, a region fails, network partitions,
a bad deploy ships. For each: how do we **detect, contain, and recover**? How do we prevent **cascading failure**?

### 16. Security
Right-size it — don't turn every interview into a security review. Prioritize: authN/authZ (OAuth/OIDC, RBAC),
TLS in transit + encryption at rest, secrets management, service-to-service auth, rate limiting, and
**tenant isolation** for multi-tenant systems. Flag PII/PCI/HIPAA only where the data demands it.

### 17. Observability
You can't operate what you can't see. Name the **metrics** (RPS, p50/p95/p99, error rate, cache hit ratio,
**queue depth**, **consumer lag**, oldest-message age, DLQ count), **structured logs**, **distributed tracing**
(trace IDs / span IDs across services), and **what triggers an alert**. Reference **RED** (Rate, Errors,
Duration) for services and **USE** (Utilization, Saturation, Errors) for resources.

### 18. Cost
Name the **major cost drivers** and which becomes expensive *first*: data transfer / CDN egress, storage,
Redis memory, DB replicas, Kafka clusters, video transcoding, workers. Then: *"what's the cheapest
architecture that still meets the SLA?"* Cost-awareness is a staff signal — infinite budget isn't real.

### 19. Alternatives & trade-offs
For every meaningful fork, **argue both sides** and then choose: SQL vs NoSQL, Kafka vs SQS, sync vs async,
push vs pull, fan-out-on-write vs on-read, strong vs eventual, cache vs DB, monolith vs microservices.
**Never say one option is simply "better."** Say: *"The trade-off is X buys us A at the cost of B; given our
dominant requirement is A, I'd choose X."*

### 20. Evolution with scale
Close by showing the system as a **living thing**, and name the bottleneck that forces each jump:

| Stage | Shape | What forced the jump |
|---|---|---|
| 1 · Small | 1 server + 1 DB | — |
| 2 · Moderate | LB + N stateless servers + cache | single server saturated; read latency |
| 3 · Large | + queue + workers + read replicas + partitioning | writes/CPU bound; sync work too slow |
| 4 · Extreme | + sharding + multi-region + event streaming | single DB/region is the ceiling |

*"I wouldn't introduce this complexity until [specific bottleneck] appears."*

---

## Time budget (a 45-minute interview)

| Minutes | Phase | Steps |
|---|---|---|
| 0–8 | Frame | 1–5 (requirements, scale) |
| 8–15 | Model | 6–8 (entities, API, data) |
| 15–22 | Design simplest | 9–10 |
| 22–35 | **Scale + failure** ← the meat | 11–15 |
| 35–42 | Deep dive the interviewer's interest | 16–20 |
| 42–45 | Trade-offs, evolution, wrap-up | 19–20 |

Let the interviewer steer. If they push on one area, **follow them there** — they're telling you where the
signal is. Cover breadth first, then go deep where asked.

---

## The mental checklist (memorize *this*, not architectures)

1. What are we building, and what are we **not**?
2. Read-heavy or write-heavy? What's the scale (order of magnitude)?
3. What needs **strong** consistency; what tolerates **eventual**?
4. What's **synchronous** (blocks the user) vs **asynchronous**?
5. What's the **dominant access pattern** → which datastore?
6. What breaks **first**, and how do I scale just that?
7. What happens when each component **fails**?
8. How do I **see** it (metrics/alerts), and what does it **cost**?
9. What would I deliberately keep **simple**?
10. What changes at **10×**? At **100×**?

→ Next: **[Capacity estimation](00-framework/capacity-estimation.md)**
