# Notification System — Trade-offs

> Dominant requirements: **reliable multi-channel delivery**, **fan-out scale**, **priority/preference correctness**,
> and **resilience to flaky providers**. Judge every trade against those.

---

## 1. Sync vs async delivery

- **Sync** (send in the caller's request): simple, immediate error to caller — but couples the caller to provider
  latency/failures and can't absorb bursts.
- **Async** (accept `202`, deliver via queue): decouples callers from flaky providers, buffers fan-out bursts, enables
  retries/DLQ. Delivery becomes eventual; caller gets an id, not a result.

**Choose:** **async** for anything at scale — the whole point is to not block callers on unreliable third parties.
(A caller that truly needs a synchronous result should call the provider directly.)

---

## 2. Single unified queue vs per-channel + per-priority queues

- **Unified:** simplest; but email/SMS/push have wildly different throughput, cost, and provider limits, and a bulk
  backlog would delay transactional.
- **Per-channel + per-priority:** each lane sized to its provider + urgency; **transactional never waits behind bulk**.

**Choose:** **separate lanes.** It's the design's backbone — priority isolation + independent scaling per channel.
See [prioritization](02-systems/notification-system/deep-dives.md#4-prioritization--fairness-transactional-vs-bulk).

---

## 3. At-least-once + dedup vs at-most-once

- **At-least-once + dedup:** never miss (critical for OTP); a duplicate is prevented by the dedup key and, worst case,
  merely annoying.
- **At-most-once:** never duplicate, but may drop — unacceptable for OTP/security.

**Choose:** **at-least-once + dedup.** Missing a notification is worse than a rare duplicate; dedup makes duplicates rare.

---

## 4. Single provider vs multi-provider

- **Single:** simpler integration + billing; but a provider outage = total channel outage, and you're at their mercy
  on rate limits/reputation.
- **Multi-provider (primary + failover):** resilience + rate-limit headroom + negotiating leverage; more integration
  work and reconciliation.

**Choose:** **multi-provider for critical channels** (email/SMS) via the channel abstraction; single is fine early.
The abstraction makes adding a failover cheap later.

---

## 5. Push token storage & pruning: eager vs lazy

- **Eager:** proactively validate tokens — extra work, cleaner data.
- **Lazy:** prune only when a provider reports "invalid token" on send — simple, self-healing.

**Choose:** **lazy pruning** driven by provider feedback (the standard) — providers tell you when a token dies.

---

## 6. Store delivery status in the hot DB vs an append store

- **Hot DB:** simple to query, but ~250 GB/day of high-write status data will crush an OLTP DB.
- **Append/columnar store + TTL:** built for high write volume + time-series reads; cheap; expire old data.

**Choose:** **append/columnar store with TTL** for status/history; keep only what the product needs hot.
See [capacity](02-systems/notification-system/capacity.md#storage).

---

## 7. Fan-out: load all recipients vs stream in batches

- **Load all:** simple but a 50M-user broadcast blows up memory and creates one giant unit of work.
- **Stream/batch:** expand the segment in bounded batches, enqueuing as you go — steady memory, natural backpressure.

**Choose:** **stream in batches.** Never materialize a 50M-row recipient list in memory.

---

## 8. Aggregation/digest vs send-every-event

- **Send every event:** immediate, but causes notification fatigue + cost + provider load.
- **Aggregate/digest:** batch similar notifications over a window → fewer, better notifications; adds delay + complexity.

**Choose:** **aggregate non-urgent, high-frequency** notifications; send **urgent** ones immediately. Product-driven;
often deferred but worth naming.

---

## The one-paragraph summary (say this)

> *"It's an async fan-out pipeline. Ingestion accepts requests fast (202), dedups by event id, and enqueues; fan-out
> workers expand to per-user/per-channel messages, enforcing preferences, quiet hours, and opt-out. I use separate
> queues per channel and per priority so an OTP never waits behind a marketing blast, and each channel worker is
> rate-limited to its provider's quota with retries, backoff, and a DLQ. A uniform channel abstraction over
> per-provider adapters lets me fail over (SES→SendGrid) and isolate provider quirks. Delivery is at-least-once with a
> dedup key so retries don't double-send. Preferences and device tokens are cached (read on every fan-out); delivery
> status goes to an append store with TTL. Big broadcasts are streamed in batches and paced to provider limits."*

→ Next: **[Failure Scenarios](02-systems/notification-system/failure-scenarios.md)**
