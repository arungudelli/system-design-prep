# Notification System — Capacity Estimation

> See the method in **[Capacity estimation](../../00-framework/capacity-estimation.md)**. For notifications the key
> numbers are **sends/sec** (per channel, since limits differ), **fan-out amplification** (one event → many sends),
> and **delivery-status write volume**. The dominant insight: a single broadcast event can create a massive,
> bursty amplification — that's the whole scaling story.

---

## Assumptions (state them)

- **50M DAU**; average **10 notifications/user/day** across channels.
- Channel mix: ~70% push, ~20% email, ~10% SMS (SMS is expensive → kept small).
- Occasional **broadcasts**: one campaign → up to **50M sends** in a short window.
- Peak ≈ 5–10× average (campaigns + event spikes).

---

## Steady-state send rate

```text
50M users × 10 notifs/day = 500M notifications/day
500M / 10^5 s ≈ 5,000 sends/sec  (average)
peak ≈ 5,000 × 10 ≈ 50,000 sends/sec
```

Split by channel (limits differ, so size each lane):
```text
push  ~70% → ~3,500/s avg
email ~20% → ~1,000/s avg
SMS   ~10% → ~500/s avg
```

→ **So what?** Each channel is a **separate queue + worker pool** sized to its own rate *and its provider's rate
limit*. Push scales cheaply; SMS is throttled by carriers/Twilio and costs real money → its lane is small and
rate-limited. You don't size one uniform pool.

---

## Fan-out amplification (the real scaling problem)

```text
A "new post" from a celebrity with 10M followers = 1 event → up to 10M notifications.
A marketing broadcast = 1 request → 50M sends.
```

→ **So what?** The system must **expand one request into millions of per-user sends** and absorb the **burst**
without falling over or blocking transactional traffic. This is why fan-out is a distinct stage feeding **buffered
queues**, and why **priority lanes** matter — a 50M broadcast must not delay an OTP. The queue is the shock absorber
(see [backpressure](../../01-patterns/queues-workers.md#backpressure-protecting-yourself-when-producers-outrun-consumers)).

---

## Provider rate limits (an external ceiling)

- Providers cap your send rate (e.g. SES/SNS/Twilio/APNs each have quotas). You **cannot** exceed them by adding
  workers — excess just gets throttled (429).
→ **So what?** Each channel worker pool is governed by a **[rate limiter](../../01-patterns/rate-limiting.md)** matched to
the provider's quota, and a big campaign is **paced** to fit within the quota over time. Throughput on a channel is
bounded by the provider, not your fleet.

---

## Storage

```text
Notification records (for dedup + status): 500M/day × ~500 bytes ≈ 250 GB/day
User preferences: 50M users × ~1 KB ≈ 50 GB (small, hot, cacheable)
Device tokens: 50M users × few devices × ~200 bytes ≈ tens of GB
Templates: tiny
```

→ **So what?** **Delivery-status/history is the size driver** (~250 GB/day) → **TTL/retention** + a cheap append
store (or object storage for old data); don't keep it forever in the hot DB. **Preferences + device tokens are
small and read on every fan-out** → cache them. Templates are trivial.

---

## Summary — what the numbers told us

| Number | Value | Design consequence |
|---|---|---|
| Send rate | ~5k/s avg, ~50k peak | Per-channel queues + worker pools sized individually |
| Fan-out | 1 event → up to 50M sends | Dedicated fan-out stage + buffered queues + **priority lanes** |
| Provider limits | external quotas | Per-channel **rate limiting**; pace big campaigns |
| Status/history | ~250 GB/day | **TTL + cheap append store**; the size driver |
| Preferences/tokens | ~50 GB, hot | **Cache** — read on every fan-out |

The profile: **fan-out + producer/consumer + flaky-downstream resilience**, moderate steady QPS with huge bursts.
Design for the burst and the priority split, not the average.

→ Next: **[API & Data Model](api-data-model.md)**
