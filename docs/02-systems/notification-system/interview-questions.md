# Notification System — Interview Questions

> Answer each **out loud first**, then expand and self-critique against the
> [critique lens](../../00-framework/staff-level-thinking.md#how-your-answer-gets-graded-the-critique-lens).

---

## Core

**Q1. What's the overall shape of this system?**
<details markdown="1"><summary>Model answer</summary>

An **async fan-out pipeline**: ingestion API (validate, dedup, `202`) → intake queue → fan-out workers (expand to
per-user/per-channel, apply preferences/quiet-hours/opt-out, render templates) → **per-channel + per-priority
queues** → channel workers → **provider adapters** → delivery tracking. Async so callers never block on flaky
providers; queues absorb fan-out bursts. See [architecture](architecture.md).
</details>

**Q2. Why separate queues per channel and per priority?**
<details markdown="1"><summary>Model answer</summary>

Channels differ in throughput, cost, and provider rate limits (SMS is slow/expensive/throttled; push is cheap/fast),
so each needs its own sized, rate-limited lane. And **transactional must not wait behind bulk** — an OTP can't queue
behind a 50M-user campaign, so transactional and bulk are **physically separate** queues/pools. See [prioritization](deep-dives.md#4-prioritization--fairness-transactional-vs-bulk).
</details>

**Q3. How do you handle unreliable third-party providers?**
<details markdown="1"><summary>Model answer</summary>

A **uniform channel abstraction** over per-provider adapters → **fail over** (SES→SendGrid, Twilio→SNS) via a
**circuit breaker** when one is down; **rate-limit** to each provider's quota; **classify errors** (throttle → pace +
requeue; 5xx → retry w/ backoff; 4xx → drop/prune); retries → **DLQ + alert**; bulkhead so one bad provider can't
exhaust shared threads. Async ingestion means a provider outage delays, not drops. See [channel abstraction](deep-dives.md#1-the-channelprovider-abstraction).
</details>

**Q4. How do you avoid sending the same notification twice?**
<details markdown="1"><summary>Model answer</summary>

**At-least-once + dedup.** Idempotency-Key (the event id) on ingestion collapses duplicate requests; a **dedup key**
`hash(user, event, channel [, window])` recorded with a TTL makes a redelivered queue message or double-emitted event
a no-op; provider idempotency as a second layer. Missing is worse than duplicating, so we choose at-least-once and
dedup the duplicates. See [dedup](deep-dives.md#3-deduplication-dont-double-notify).
</details>

---

## Fan-out & scale

**Q5. A celebrity with 10M followers posts, or marketing broadcasts to 50M users. Walk the fan-out.**
<details markdown="1"><summary>Model answer</summary>

Ingestion accepts the single request (`202`) and enqueues. Fan-out workers **stream the recipient list in batches**
(never load 50M into memory), and per user apply preferences/opt-out/quiet-hours, render the template, and enqueue one
message per (user, channel) to the **bulk** channel queues. Bulk workers drain at the provider's rate limit, **paced**
over time. The queues buffer the burst; transactional lanes are untouched. See [capacity](capacity.md#fan-out-amplification-the-real-scaling-problem).
</details>

**Q6. How do you make sure an OTP arrives in seconds while a huge campaign is running?**
<details markdown="1"><summary>Model answer</summary>

**Separate transactional and bulk lanes** (distinct queues + dedicated worker capacity) per channel. The transactional
queue is short and drains immediately; the campaign sits in the bulk queue and can't block it. Transactional/security
also bypass quiet hours and opt-out. Physical separation, not just a priority flag, is what guarantees it.
</details>

**Q7. What's the first bottleneck as you scale, and how do you address it?**
<details markdown="1"><summary>Model answer</summary>

**Fan-out amplification** (1 event → millions of sends, bursty) and **provider rate limits**. Address fan-out by
streaming recipient expansion in batches + autoscaling fan-out workers + buffering in queues; address provider limits
with per-channel rate limiters, campaign pacing, and multi-provider failover. Then cache preferences/tokens (read on
every send) and offload status writes to an append store.
</details>

---

## Preferences & compliance

**Q8. How and where do you enforce user preferences, quiet hours, and opt-out?**
<details markdown="1"><summary>Model answer</summary>

At **fan-out**, before any send: per-channel/per-category opt-in/out, quiet hours in the user's timezone, and global
unsubscribe. **Precedence:** security/transactional > category prefs > quiet hours > channel prefs; marketing never
overrides quiet hours; security/OTP usually isn't opt-out-able. Opt-out must be **immediate and auditable** (legal:
CAN-SPAM/TCPA). Preferences are cached but invalidated promptly on change. See [preferences](deep-dives.md#2-user-preferences-quiet-hours--opt-out-product--legal).
</details>

**Q9. How do you protect email sender reputation?**
<details markdown="1"><summary>Model answer</summary>

Consume provider webhooks: on **hard bounce or spam complaint**, **auto-suppress** that address so you never send to
it again; monitor bounce/complaint rates; keep lists clean (lazy-prune invalid addresses/tokens). Reputation is a
shared fragile resource — bad addresses and complaints get you throttled or blocked by the provider.
</details>

---

## Reliability

**Q10. How do retries work, and how do you avoid retrying things that will never succeed?**
<details markdown="1"><summary>Model answer</summary>

**Classify errors:** throttle/429 → pace + requeue (not a failure); transient 5xx/timeout → **retry with backoff +
jitter**, capped; permanent 4xx (invalid address/token, unsubscribed) → **don't retry**, drop + prune. After max
retries → **DLQ + alert**, redrive after fixing. A **circuit breaker** fails a broadly-down provider over to the
secondary. See [retries](deep-dives.md#5-retries-dlq--failing-providers).
</details>

**Q11. Should ingestion fail if a provider is down?**
<details markdown="1"><summary>Model answer</summary>

**No.** Ingestion just validates, dedups, persists, and enqueues — it returns `202` regardless of provider health.
The queue buffers; delivery happens when the provider recovers (or via failover). Callers must never fail because a
downstream provider is having a bad day. Delayed, not dropped.
</details>

---

## Staff-level curveballs

**Q12. How do you fight notification fatigue?**
<details markdown="1"><summary>Model answer</summary>

**Aggregation/digest** (batch "10 people liked your post" over a window instead of 10 notifications), **per-user rate
caps** (max N/day), respecting **categories** so users get only what they want, and **quiet hours**. Aggregation uses
a short buffering window per (user, event-type) or a periodic digest job (via the [job scheduler](../job-scheduler/README.md)).
Product-driven; worth naming even if deferred.
</details>

**Q13. Do you need delivery/read receipts, and what do they cost?**
<details markdown="1"><summary>Model answer</summary>

Delivery tracking (provider webhooks → status store) is valuable for analytics, bounce handling, and failover signals,
but it's a **large write path** (one+ event per send, ~250 GB/day at our scale). Keep it **off the critical send
path** (async), store in an append/columnar store with **TTL**, and don't let a slow status store block delivery.
Read receipts (opened) add more volume — include only if the product needs them.
</details>

**Q14. Where does this reuse patterns you've studied?**
<details markdown="1"><summary>Model answer</summary>

**Queue + workers:** the whole pipeline (intake, fan-out, per-channel lanes). **Idempotency:** dedup keys prevent
double-sends. **Rate limiting:** per-provider quotas + per-user fatigue caps. **Circuit breaker/bulkhead:** provider
resilience. **Caching:** preferences + device tokens. **Sharding:** status/feed by user. **Job scheduler:** scheduled
sends + digests. It's a composition of the pattern library around flaky downstreams.
</details>

**Q15. What changes at 10× and 100×?**
<details markdown="1"><summary>Model answer</summary>

**10×:** more fan-out + channel workers; multi-provider per channel for rate-limit headroom; harder campaign pacing.
**100×:** multi-region ingestion/queues, sharded status store with aggressive TTL, dedicated digest/aggregation to cut
volume, and careful provider-reputation + quota management across many providers. The async fan-out + separate-lanes
backbone holds; the pressure is on fan-out throughput, provider limits, and status-write volume.
</details>

**Q16. When would you redesign this system?**
<details markdown="1"><summary>Model answer</summary>

When assumptions break: notifications need **real-time interactive delivery** (lean harder on WebSocket/in-app +
presence), volume/fatigue demands a **first-class aggregation/ML-ranked feed** (a feed system, not a notifier),
compliance regimes multiply (regional routing + consent as a core service), or scale jumps 100× (multi-region,
multi-provider orchestration). Absent those, the pipeline holds.
</details>

---

## Self-scoring

Grade against the [15-point critique lens](../../00-framework/staff-level-thinking.md#how-your-answer-gets-graded-the-critique-lens):
did you make it async with buffered ingestion, separate channel + priority lanes, abstract providers with failover,
dedup for exactly-once effect, enforce preferences/opt-out at fan-out, and handle provider throttling/outages? Note
your weakest area and drill it.

← Back to **[README](README.md)** · Next: **[Cheatsheet](cheatsheet.md)**
