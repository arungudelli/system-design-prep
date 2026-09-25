# Notification System

> Design a system that delivers notifications to users across **multiple channels** — push (iOS/Android), SMS,
> email, in-app — triggered by events across the company (order shipped, friend request, price alert, OTP code),
> at scale, respecting user preferences, without spamming or double-sending.

The naive version ("call SendGrid in the request handler") is trivial; the interview is about everything around
it: **fan-out** to millions, a **channel/provider abstraction** over unreliable third parties, **user preferences +
quiet hours + opt-outs**, **deduplication**, **retries/DLQ per channel**, **rate limiting**, **prioritization**
(an OTP must beat a marketing blast), and **delivery tracking**. It's a natural application of everything in the
[queues + workers](../../01-patterns/queues-workers.md), [idempotency](../../01-patterns/idempotency.md), and
[rate limiting](../../01-patterns/rate-limiting.md) patterns.

---

## Why it's a great system to study

- **Producer/consumer + fan-out at its core** — many event sources, many channels, many providers.
- **Third-party providers are unreliable** — SES/SNS/SendGrid/Twilio/APNs/FCM fail, throttle, and rate-limit you →
  the design is really about **isolating and retrying around flaky downstreams**.
- **Correctness meets product** — dedup, preferences, quiet hours, and compliance (opt-out/CAN-SPAM/TCPA) are as
  important as throughput.
- **Prioritization + fairness** — transactional (OTP, password reset) vs bulk (marketing) can't share one lane.

---

## Read in this order

1. **[Requirements](requirements.md)** — problem, clarifying questions, functional + NFRs
2. **[Capacity](capacity.md)** — notifications/sec, fan-out, storage
3. **[API & Data Model](api-data-model.md)** — send API, templates, preferences, device tokens
4. **[Architecture](architecture.md)** — ingestion → fan-out → channel workers → providers
5. **[Deep Dives](deep-dives.md)** — channel abstraction, preferences, dedup, prioritization, retries, delivery tracking
6. **[Trade-offs](tradeoffs.md)** — the decisions, both sides
7. **[Failure Scenarios](failure-scenarios.md)** — provider outage, poison messages, dup sends
8. **[Interview Questions](interview-questions.md)** — attempt first, then reveal
9. **[Cheatsheet](cheatsheet.md)** — 1-page revision
10. **[★ Principal Deep Dive](principal-deep-dive.md)** — the **staged stops** (inline send → global) + max-scale variant (multi-provider, multi-region, delivery-tracking stream, ML/aggregation) with a reconciliation table of when to adopt/remove each piece

---

## The 30-second version (know this cold)

- **Pipeline:** event → **ingestion API** (validate, dedup, resolve recipients) → **queue** → **fan-out** (expand to
  each channel per user preferences) → **per-channel queues + workers** → **provider adapters** (email/SMS/push) →
  **delivery tracking**.
- **Separate queues per channel** (and per priority) — email throughput ≠ SMS throughput ≠ push; an OTP must not
  wait behind a 10M-user marketing blast → **priority lanes**.
- **Provider abstraction:** a common `Channel.send()` interface with adapters per provider (SES, Twilio, APNs, FCM),
  so you can fail over between providers and isolate their quirks/rate limits.
- **User preferences + quiet hours + opt-out** are checked at fan-out — the system must never send what a user
  disabled (also a legal requirement).
- **Idempotency/dedup:** at-least-once queues + a dedup key (`user + event + channel + window`) so retries and
  duplicate events don't double-notify.
- **Retries/DLQ per channel** with backoff; respect provider throttles (rate limiting); circuit-break a failing provider.
- **Bottlenecks:** fan-out amplification (1 event → millions of sends), provider rate limits, and the delivery-status
  write volume.
