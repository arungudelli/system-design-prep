# Interview Wording

> Same design, two candidates. One sounds like they've operated systems in production; the other sounds
> like they read a blog. The difference is **phrasing that signals judgment** — clear, calm, buzzword-free.
> These are reusable lines. Internalize the *shape*, not the exact words.

Rule of thumb: **narrate your reasoning, not just your conclusion.** "I'll add a cache" is a conclusion.
"Reads are 100× writes, so the read path is the bottleneck — I'll add a cache-aside layer keyed by X" is judgment.

---

## Opening — framing (buy yourself thinking time, show discipline)

> "Before I design anything, I'd like to clarify the requirements and agree on scope."
> "Let me separate must-haves from nice-to-haves so we don't over-scope."
> "I'll explicitly defer analytics and custom aliases for now, and come back if we have time."
> "Let me confirm the dominant non-functional requirement — I'm hearing this is read-heavy and latency-sensitive. Is that right?"
> "Let me estimate the order of magnitude before committing to an architecture."

## Estimation

> "Seconds in a day is roughly 10⁵, so that's about N requests per second on average."
> "I'll assume peak is about 3× average unless you'd tell me it's spikier."
> "The read:write ratio is about 100:1, so reads are the problem I need to design for."
> "That number is small enough that a single database handles it — I won't shard prematurely."

## Choosing simple (the most underrated staff signal)

> "I'll start with the simplest architecture that satisfies these requirements, then evolve it as we find bottlenecks."
> "I don't think we need a message queue yet — a synchronous call is simpler and meets the latency budget. I'd add async only when [specific reason]."
> "Before introducing sharding, I'd first exhaust indexing, caching, and read replicas."
> "I'd keep this a modular monolith to start; I'd split out a service only when a part needs independent scaling or deployment."
> "I could use Kafka here, but SQS meets our needs with far less to operate. I'd switch to Kafka if we needed replay or high fan-out."

## Data & consistency

> "Let me ask what the dominant access patterns are before I pick a database."
> "This access pattern is a simple key lookup, so a key-value store fits better than a relational schema."
> "Payments need strong consistency and a hard durability guarantee. Follower counts can be eventually consistent."
> "The user needs read-your-writes here, so I'll route their reads to the primary for a short window after a write."
> "I'll put a unique constraint on this column so a retried request can't create a duplicate."

## Scaling (identify → justify → scale one thing)

> "This service is stateless, so I can scale it horizontally behind the load balancer and autoscale on CPU."
> "I'll push session state into Redis so any server can handle any request."
> "The primary bottleneck here is likely the database write path — I'd detect it via rising write latency and replication lag."
> "At 10× this holds with more replicas; at 100× the single primary is the ceiling, and that's when I'd shard by [key]."
> "I'd shard on [key] because it spreads load evenly and keeps the common query on a single shard."

## Async & reliability

> "This work doesn't need to block the user, so I'd move it to a queue and process it asynchronously."
> "I'll use at-least-once delivery with idempotent consumers — true exactly-once end-to-end is impractical."
> "To make retries safe, the consumer dedupes on an idempotency key before applying the effect."
> "I'll wrap the downstream call in a timeout and a circuit breaker so a slow dependency can't exhaust our threads."
> "Failed messages retry with exponential backoff and jitter; after N attempts they go to a dead-letter queue and we alert."
> "To avoid the write-then-publish gap, I'd use a transactional outbox instead of a distributed transaction."

## Failure analysis (say this proactively — don't wait to be asked)

> "Let me walk through the failure scenarios."
> "If the cache dies, we fall back to the database in a degraded mode, and I'd guard the refill against a stampede."
> "If the primary fails over, writes pause briefly; reads continue from replicas. RTO is a few seconds."
> "If an entire AZ fails, we're multi-AZ so we lose capacity but not availability."
> "I'd only go multi-region if the RTO/RPO or latency requirements justified the complexity and cost."
> "To prevent cascading failure, I isolate this dependency behind a bulkhead with its own connection pool."

## Trade-offs (always two-sided, then commit)

> "The main trade-off here is X versus Y. X gives us [benefit] at the cost of [downside]."
> "Given our dominant requirement is [latency / durability / cost], I'd choose X — but if [assumption] changed, Y would win."
> "Fan-out-on-write makes reads cheap but writes expensive; for a celebrity account I'd switch to fan-out-on-read. A hybrid handles both."
> "I wouldn't call one strictly better — it depends on whether we optimize for read latency or write cost."

## Observability & cost

> "I'd measure RPS, p99 latency, error rate, and — for the async path — queue depth and consumer lag."
> "I'd alert on p99 breaching the SLA and on the oldest-message age crossing a threshold, not just on raw error count."
> "The first thing to get expensive here is [CDN egress / Redis memory / DB replicas]; I'd watch that cost line closely."

## Closing / wrap-up

> "To summarize: simplest design is A; the first bottleneck is B, which I scale with C; the main risks are D, mitigated by E."
> "I'd keep [these parts] deliberately simple, and only invest complexity in [the critical path]."
> "I'd redesign this when [specific trigger] — and here's the migration path I'd take."

---

## Phrases to **avoid** (buzzword tells)

| Don't say | Say instead |
|---|---|
| "Just use Kafka, it's more scalable." | "SQS is enough here; Kafka only earns its complexity if we need replay or high fan-out." |
| "We'll make it exactly-once." | "At-least-once with idempotent consumers." |
| "Add a cache" (no reason) | "Reads are 100× writes, so I'll cache the read path keyed by X." |
| "Use microservices." | "I'd keep a modular monolith and split out a service only when a part needs independent scaling." |
| "It's web-scale." | *(give the actual RPS / storage number)* |
| "NoSQL is faster." | "This access pattern is a key lookup, so a KV store fits; a relational store would over-serve it." |

---

## Meta-tips on delivery

- **Think out loud.** Silence reads as being stuck. Narrate even the discarded options.
- **Drive, but let them steer.** Cover breadth, then go deep where the interviewer leans in.
- **Ask before assuming**, then state your assumption and move — don't stall waiting for permission.
- **Admit unknowns cleanly:** "I haven't operated X at this scale, but here's how I'd reason about it."
  Honesty + a reasoning path beats a confident wrong answer.
- **Manage the clock.** If you're deep in the weeds at minute 35, zoom back out to trade-offs and evolution.

→ Next: **[Interview checklist](interview-checklist.md)**
