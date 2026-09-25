# Notification System — Cheatsheet

> One-page revision. Reproduce from memory and you can drive the interview.

---

## Problem
Deliver notifications across push / SMS / email / in-app, triggered by events, at scale, honoring preferences, no spam/dupes.

## Profile (say first)
**Async fan-out + producer/consumer + flaky-downstream resilience.** Moderate steady QPS, huge bursty fan-out.

## Requirements
- **Must:** accept requests, multi-channel delivery, honor preferences/opt-out, reliable (retry/DLQ/dedup), provider integration.
- **Defer:** delivery tracking detail, templates/i18n detail, digests, scheduling (→ job scheduler).
- **Dominant NFRs:** reliable multi-channel · fan-out scale · priority/preference correctness · provider resilience.

## Numbers (example)
| | |
|---|---|
| Sends | 50M DAU × 10/day = 500M/day → ~5k/s avg, ~50k peak |
| Channel mix | push 70% / email 20% / SMS 10% (SMS = expensive, throttled) |
| Fan-out | 1 broadcast → up to **50M sends** (the scaling problem) |
| Status/history | ~250 GB/day → **TTL + append store** (size driver) |
| Prefs/tokens | small, hot → **cache** |

## Architecture
```text
services → [Ingestion API] validate·dedup(event_id)·persist → 202
             │ enqueue
        [Intake Queue]
             │
     [Fan-out workers] expand per-user · prefs/quiet-hours/opt-out · render template
             │  route by CHANNEL + PRIORITY
     ┌───────┼───────┬───────┐
  push q   email q  sms q   in-app     (×2: transactional | bulk lanes)
     │       │       │        │
  workers (rate-limited to provider quota · retry/backoff · circuit breaker)
     │
  Provider adapters: APNs/FCM · SES/SendGrid · Twilio/SNS
     │ webhooks
  [Delivery tracking] → status store · prune dead tokens · suppress bounces
```

## Key ideas
- **Async (202)** — never block callers on flaky providers; queues buffer bursts.
- **Separate lanes per channel + priority** — OTP never waits behind a 50M campaign.
- **Channel abstraction** — uniform `send()` + per-provider adapters → **failover** (SES→SendGrid) + isolate quirks.
- **Preferences/quiet-hours/opt-out at fan-out** — precedence: security > category > quiet hours > channel. Opt-out
  immediate + auditable (CAN-SPAM/TCPA).
- **Dedup:** at-least-once + `hash(user,event,channel)` key + idempotency-key on ingestion → no double-send.
- **Stream fan-out in batches** — never load 50M users into memory.
- **Pace campaigns** to provider rate limits; **per-user caps** for fatigue.

## Retries / providers
Classify errors: **429 → pace + requeue** · **5xx → retry backoff+jitter** · **4xx → drop + prune**. → DLQ + alert.
Circuit breaker → failover to secondary provider. Bounce/complaint → **auto-suppress** address.

## Bottlenecks (ranked)
1. **Fan-out amplification** → stream batches + buffer + autoscale
2. **Provider rate limits** → per-channel limiter + pacing + multi-provider
3. **Priority inversion** → separate transactional/bulk lanes
4. Prefs/device reads → cache
5. Status writes → append store + TTL

## Failure one-liners
- Provider down → circuit breaker → failover / buffer (delayed, not lost).
- 429 → slow lane + requeue (not a failure). Bulk starves OTP → separate lanes.
- Dup → dedup key + idempotency. Opt-out send → enforce at fan-out (legal).
- Poison → DLQ. Dead token → prune. Hard bounce → suppress.
- Ingestion never fails on provider outage (202 + buffer).

## Patterns used
[queue+workers](../../01-patterns/queues-workers.md) · [idempotency](../../01-patterns/idempotency.md) ·
[rate limiting](../../01-patterns/rate-limiting.md) · circuit breaker/bulkhead · [caching](../../01-patterns/caching.md) ·
[sharding](../../01-patterns/sharding.md) · [job scheduler](../job-scheduler/README.md) (scheduled/digests).

## SQL vs NoSQL
Prefs/devices/templates → KV/relational + cache. Status/feed → wide-column sharded by user + TTL.

## Summary line
> "Async fan-out pipeline: ingestion dedups + 202; fan-out expands per-user applying prefs/quiet-hours/opt-out;
> separate per-channel + per-priority queues so OTP beats bulk; channel workers rate-limited to provider quotas with
> retries/DLQ + circuit-breaker failover across a uniform provider abstraction. At-least-once + dedup key = no
> double-send. Stream broadcasts in batches, pace to provider limits, cache prefs/tokens, TTL the status store."

← Back to **[README](README.md)**
