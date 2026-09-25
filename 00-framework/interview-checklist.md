# Interview Checklist

> The one-page scan. Read this on the way into the interview. It's the **[20-step framework](00-framework/system-design-framework.md)**
> compressed to what you can hold in your head.

---

## The 6 questions that must be answered in the first 8 minutes

1. **What are we building** — and what are we explicitly **not** building?
2. **Read-heavy or write-heavy?** What's the **read:write ratio**?
3. **Scale** — order of magnitude of DAU, peak RPS, storage/year?
4. What needs **strong consistency**; what tolerates **eventual**?
5. What's **synchronous** (user waits) vs **asynchronous**?
6. What's the **dominant NFR** (latency? durability? availability? cost?)

If you can answer these six, the architecture almost designs itself.

---

## The flow (glance, don't recite)

```text
FRAME      1 clarify → 2 functional (must vs nice) → 3 NFRs (quantified)
           4 assumptions → 5 estimate (peak RPS, storage, bandwidth)
MODEL      6 entities → 7 APIs → 8 data model (access patterns FIRST → DB choice)
DESIGN     9 simplest architecture (justify every box) → 10 walk read/write/async/failure flows
SCALE      11 rank bottlenecks (why/detect/metric/fix) → 12 scale each (cache→replicas→shard)
HARDEN     13 reliability → 14 consistency → 15 failure scenarios
           16 security → 17 observability → 18 cost
CLOSE      19 trade-offs (both sides) → 20 evolution (stage 1→4, name each trigger)
```

---

## Don't reach for these unless a requirement forces it

`Kafka` · `Kubernetes` · `Cassandra` · `sharding` · `microservices` · `multi-region`

For each one you *do* add, say: **"why do we need this, and what does it solve that the simpler design can't?"**

---

## Scaling ladder (cheapest first — always in this order)

```text
1. Vertical scale / tune queries + indexes   (cheapest, often enough)
2. Stateless app tier + load balancer + autoscale
3. Cache the read path  (cache-aside)
4. Read replicas
5. Async: move slow work to queue + workers
6. Partition
7. Shard        ← last resort: hot shards, resharding, cross-shard queries
8. Multi-region ← only if RTO/RPO/latency demands it
```

Say it: *"Before sharding I'd first exhaust indexing, caching, and read replicas."*

---

## The failure sweep (walk these proactively)

| Component | What happens | Mitigation |
|---|---|---|
| API server dies | LB routes around it | stateless + health checks |
| Worker dies mid-job | job redelivered | idempotent consumer, visibility timeout |
| DB primary fails | brief write pause | failover, reads from replicas |
| Cache dies | load hits DB | degrade gracefully, stampede guard |
| Queue unavailable | producers block | bounded buffer, backpressure, shed load |
| Downstream slow (200ms→10s) | threads pile up | timeout + circuit breaker + bulkhead |
| AZ fails | lose capacity | multi-AZ |
| Region fails | lose region | multi-region (if justified) |
| Bad deploy | errors spike | canary, rollback, backward-compatible schema |

---

## Reliability toolkit (name-drop with intent)

`timeout` · `retry + exponential backoff + jitter` · `idempotency key` · `circuit breaker` · `bulkhead` ·
`DLQ` · `backpressure / load shedding` · `graceful degradation` · `transactional outbox`

**Mantra:** at-least-once delivery + **idempotent consumers**. Never promise end-to-end exactly-once.

---

## Observability (what to measure & alert on)

- **Metrics:** RPS · p50/p95/**p99** · error rate · cache hit ratio · **queue depth** · **consumer lag** · oldest-message age · DLQ count
- **Alert on:** p99 breaching SLA · error-rate spike · oldest-message age · DLQ growth (symptoms, not just resources)
- **Also:** structured logs · distributed tracing (trace/span IDs) · **RED** (services) / **USE** (resources)

---

## Trade-offs you should be able to argue *both sides* of

SQL ↔ NoSQL · Kafka ↔ SQS ↔ RabbitMQ · sync ↔ async · push ↔ pull ·
fan-out-on-write ↔ on-read · strong ↔ eventual consistency · cache ↔ DB ·
monolith ↔ microservices · precompute ↔ compute-on-read · WebSocket ↔ SSE ↔ polling ·
single-region ↔ multi-region · active-active ↔ active-passive

Never declare a winner without naming the trade and tying it to the dominant requirement.

---

## Final self-check (before you say "I'm done")

Can I answer, for *this* design:

- [ ] What are we building, who uses it, at what scale?
- [ ] Read/write pattern, and what data we store?
- [ ] What needs strong vs eventual consistency?
- [ ] What's sync vs async?
- [ ] What bottlenecks first, and how I scale it?
- [ ] What happens when each part fails?
- [ ] How I observe it and secure it?
- [ ] What it costs, and what I'd simplify?
- [ ] What changes at **10×** and **100×**?

---

## Anti-patterns that cost staff-level points

- Drawing boxes before clarifying requirements
- Adding components you can't justify ("resume-driven design")
- Designing only the happy path
- Promising exactly-once delivery
- Precision theatre in estimates (5 sig figs of a guessed input)
- Declaring a tech "better" without a trade-off
- Sharding / microservices / multi-region by reflex
- Going silent — think out loud

← Back to **[The framework](00-framework/system-design-framework.md)**
