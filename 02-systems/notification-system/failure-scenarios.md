# Notification System — Failure Scenarios

> Themes: **third-party providers are unreliable** (the whole system is built to survive that), **never let bulk
> starve transactional**, **never double-send or send after opt-out**, and **accept ingestion even when downstream is
> down**.

---

## A provider goes down (e.g. SES / Twilio outage)

- **Impact:** that channel can't deliver via the primary provider.
- **Detect:** spike in errors/timeouts from the provider adapter; delivery rate drops.
- **Contain:** **circuit breaker** trips → **fail over to the secondary provider** via the channel abstraction; messages
  keep flowing. If no secondary, messages **stay queued** (buffered) and drain when the provider recovers — delayed,
  not lost.
- **Recover:** breaker half-opens to test recovery; resume primary. Backlog drains at rate-limited pace.

## A provider throttles us (429)

- **Impact:** sends rejected for exceeding quota.
- **Detect:** 429 rate.
- **Contain:** this is **not a failure** — **slow the lane** to the provider's rate limit and **requeue**; pace
  campaigns. Adding workers won't help (the ceiling is external). Optionally spread load across a second provider.

## Bulk campaign starves transactional (priority inversion)

- **Impact:** OTPs delayed behind a 50M-user blast → users can't log in.
- **Contain:** **separate queues/pools for transactional vs bulk** per channel → the transactional lane is short and
  drains fast regardless of the bulk backlog. This is why lanes are physically separate, not just prioritized flags.

## Duplicate sends

- **Impact:** user gets the same notification twice (annoying; for OTP, confusing).
- **Cause:** at-least-once queue redelivery, retried events, double-emitted events.
- **Contain:** **dedup key** (`user+event+channel`, TTL) + **idempotency key on ingestion** + provider idempotency
  where available → a retry within the window is a no-op. See [dedup](02-systems/notification-system/deep-dives.md#3-deduplication-dont-double-notify).

## Sending after opt-out / during quiet hours (compliance failure)

- **Impact:** legal + trust violation (CAN-SPAM/TCPA).
- **Contain:** enforce preferences/opt-out/quiet-hours **at fan-out**, read fresh (cache with prompt invalidation on
  preference change). Opt-out must be **immediate and auditable**. Security/transactional override quiet hours by
  policy, never marketing.

## Poison message (bad template/payload)

- **Impact:** a message fails every attempt, risks jamming a lane.
- **Contain:** cap retries → **DLQ + alert**; **redrive** after fixing. Don't loop forever on a permanent failure.

## Invalid / stale push token

- **Impact:** push to a dead token fails.
- **Contain:** treat provider "unregistered/invalid" as **permanent (4xx) — don't retry**; **prune the token** from
  `devices`. Lazy pruning driven by provider feedback keeps the token store clean.

## Hard bounce / spam complaint (email)

- **Impact:** sending to bad addresses / complainers hurts sender reputation → providers may block you.
- **Contain:** **auto-suppress** the address on hard bounce/complaint (via the delivery-tracking webhook) so you never
  send to it again. Reputation is a shared, fragile resource.

## Ingestion overwhelmed by a fan-out burst

- **Impact:** a broadcast or event storm floods intake.
- **Contain:** ingestion just **accepts + enqueues** (cheap); the **queue buffers** the burst; fan-out workers
  autoscale on queue depth. **Stream** recipient expansion in batches (never load 50M users at once). Backpressure/load
  shedding on non-critical intake if truly overwhelmed — but transactional intake stays protected.

## Downstream status store slow/unavailable

- **Impact:** can't record delivery status.
- **Contain:** status writes are **off the critical send path** — buffer/queue them; a slow status store must **not**
  block or fail actual delivery. Worst case we lose some status granularity, not deliveries.

## Preference/device cache stale or down

- **Impact:** fan-out can't read prefs → risk sending wrong things or stalling.
- **Contain:** cache falls through to the source store (degrade); **fail safe on opt-out** — if you truly can't confirm
  consent for a *marketing* message, don't send it; for transactional/security, proceed. Invalidate cache on
  preference change so opt-outs take effect promptly.

## Region failure

- **Impact:** lose capacity/queues in a region.
- **Contain:** multi-region ingestion + queues; providers are global; the durable queue means accepted notifications
  aren't lost. Failover routing for ingestion.

---

## Failure-handling toolkit used here

`async ingestion (202) + buffered queues` (accept even when downstream down) · `circuit breaker + multi-provider
failover` · `provider rate limiting + pacing` (429) · `separate transactional/bulk lanes` (priority) · `dedup key +
idempotency` (no double-send) · `preference/opt-out enforced at fan-out` (compliance) · `retry+backoff → DLQ` (poison)
· `error classification` (throttle vs transient vs permanent) · `lazy token pruning + bounce suppression` · `stream
fan-out in batches` · `status writes off the critical path`.

## Priorities (say this)

> *"My guarantees: never miss a transactional notification, never double-send, and never send after opt-out. Async
> ingestion + buffered queues absorb provider outages (delayed, not lost); circuit breakers + multi-provider failover
> keep channels alive; separate transactional/bulk lanes keep OTPs fast during campaigns; dedup keys prevent
> duplicates; and preference/opt-out checks at fan-out keep us compliant. Everything degrades to delayed, not lost —
> except messages a user opted out of, which we must never send."*

→ Next: **[Interview Questions](02-systems/notification-system/interview-questions.md)**
