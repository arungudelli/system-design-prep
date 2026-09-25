# Notification System — Deep Dives

The interview lives here: **channel/provider abstraction**, **preferences & quiet hours**, **dedup**,
**prioritization**, **retries around flaky providers**, and **delivery tracking**.

---

## 1. The channel/provider abstraction

Third-party providers (SES, SendGrid, Twilio, SNS, APNs, FCM) each have different APIs, auth, payload formats, rate
limits, and failure modes. Wrap them behind a **uniform interface**:

```text
interface Channel { send(rendered, address, idempotencyKey) -> {status, providerMsgId} | error }
EmailChannel  → SES  (primary) | SendGrid (failover)
SmsChannel    → Twilio (primary) | SNS (failover)
PushChannel   → APNs (iOS) | FCM (Android/web)
InAppChannel  → write to feed store (no external provider)
```

- **Failover:** if the primary email provider is down/throttling, route to the secondary — the caller (channel
  worker) doesn't care which provider handled it. This is the main lever for surviving provider outages.
- **Isolation:** each provider's quirks (rate limits, retry semantics, token formats, error codes) live in its
  adapter, not smeared through the system.
- **Idempotency at the provider:** pass an idempotency key to providers that support it so a retry doesn't double-send.

> **Interview line:** *"I put a uniform Channel interface over per-provider adapters, so I can fail over from SES to
> SendGrid without touching the rest of the system, and each provider's rate limits and quirks stay isolated in its
> adapter."*

---

## 2. User preferences, quiet hours & opt-out (product + legal)

Checked at **fan-out**, before any send:

- **Per-channel, per-category** opt-in/out: a user may allow push for order updates but no marketing email.
- **Quiet hours** (in the user's **timezone**): defer or drop non-urgent notifications at night. Transactional/
  security (OTP, fraud alert) **override** quiet hours and are usually **not opt-out-able**.
- **Global opt-out / unsubscribe:** legally required (CAN-SPAM for email, TCPA for SMS). Must be honored immediately
  and be **auditable** (prove you didn't send after opt-out).
- **Precedence:** security/transactional > user category prefs > quiet hours > channel prefs. Define the order.

> **Interview line:** *"Preferences, quiet hours, and opt-out are enforced at fan-out. Security and transactional
> messages override quiet hours; marketing never does. Opt-out is immediate and auditable because it's a legal
> requirement, not just a preference."*

---

## 3. Deduplication (don't double-notify)

Duplicates come from at-least-once queues, retried events, or the same logical event emitted twice.

- **Dedup key:** `hash(user_id, event_id, channel [, time_window])`. Record it (dedup table / Redis SETNX with TTL);
  if seen, skip the send. This is [idempotency](01-patterns/idempotency.md) applied to notifications.
- **Idempotency-Key on ingestion** (the event id) collapses duplicate *requests* up front.
- **Provider idempotency** as a second layer for providers that support it.
- **Window matters:** for "you have a new message," a short window dedups a retry but still allows the *next* real
  message; for OTP, dedup on the specific code/event so the same code isn't sent twice but a resend of a *new* code works.

> **Interview line:** *"I dedup on user+event+channel with a TTL, backed by an idempotency key on ingestion, so a
> retried event or a redelivered queue message never double-notifies — while genuinely new events still go through."*

---

## 4. Prioritization & fairness (transactional vs bulk)

The signature requirement: an **OTP must arrive in seconds** even while a **50M-user campaign** is being sent.

- **Separate lanes:** distinct queues (and often distinct worker pools) for **transactional** vs **bulk** per channel.
  Transactional gets dedicated/priority capacity; bulk drains from what's left.
- **Never let bulk starve transactional:** because they're physically separate queues, a huge campaign backlog in the
  bulk queue can't delay the short transactional queue.
- **Per-tenant / per-user rate limiting** so one sender's burst can't monopolize a shared channel
  ([rate limiting](01-patterns/rate-limiting.md)); and to prevent **notification fatigue** (cap notifications/user/day).
- **Pacing campaigns:** bulk sends are paced to the provider's rate limit and spread over time (also avoids the
  thundering-herd + provider throttling).

> **Interview line:** *"Transactional and bulk go in separate queues per channel so an OTP can't get stuck behind a
> campaign. Bulk is paced to the provider's rate limit, and I cap per-user notification volume to avoid fatigue."*

---

## 5. Retries, DLQ & failing providers

Providers fail, throttle, and time out constantly — resilience is the core job.

- **Retry with backoff + jitter**, capped; after N attempts → **DLQ + alert** (and mark the notification failed).
- **Distinguish error types:** `429`/throttle → slow the lane + requeue (not a "failure"); `5xx`/timeout → retry;
  `4xx` like invalid address/token → **don't retry**, prune the token/mark bad. Retrying a permanent failure is waste.
- **Circuit breaker** per provider: if a provider is broadly failing, stop hammering it and **fail over** to the
  secondary; bulkhead so one bad provider doesn't exhaust shared threads.
- **Poison messages** (bad payload/template) → DLQ, don't loop forever.
- **At-least-once + dedup** means a retry is safe (won't double-send within the dedup window).

> **Interview line:** *"I classify provider errors: throttle → pace and requeue, transient 5xx → retry with backoff,
> permanent 4xx → drop and prune. A circuit breaker fails a broadly-down provider over to the secondary, and poison
> messages go to a DLQ so they don't jam the lane."*

---

## 6. Delivery tracking & feedback loop

- **Provider webhooks/callbacks** report `delivered`, `bounced`, `opened`, `failed` asynchronously → update the
  notification's status. This is a **large write path** (one+ event per send) → append store + TTL.
- **Bounce/complaint handling:** a hard bounce or spam complaint should **auto-suppress** future sends to that
  address (protects sender reputation; required by email providers).
- **Invalid push tokens:** APNs/FCM tell you when a token is dead → **prune it** from `devices`.
- **Metrics:** delivery rate, bounce rate, per-provider latency/error → feed provider failover + alerting.

---

## 7. Templates & localization

- **Template service:** store templates per `(template_id, channel, locale)`, versioned. Render `data` into the
  template at fan-out. Different channels need different renders (SMS = 160 chars; email = HTML; push = title+body).
- **Localization:** pick locale from user preference; fall back to default. Keep templates out of code so
  non-engineers can edit and you can change copy without a deploy.

---

## 8. In-app notifications (the channel with no external provider)

- In-app = a **write to a feed store** keyed by `user_id`, read by the client (poll, or push via WebSocket/SSE —
  see [realtime](01-patterns/websockets-realtime.md) *(coming)*). No third-party provider, so it's the most reliable
  channel and a good fallback.
- Often paired with **read/seen tracking** and **aggregation** ("3 people liked your post") to reduce noise.

---

## 9. Aggregation / digest (fighting notification fatigue)

- Instead of 10 separate "X liked your post," **batch** them into one digest ("10 people liked your post") over a
  window. Reduces spam, cost, and fatigue.
- Implemented as a short **buffering window** per (user, event-type) before fan-out, or a periodic digest job (via the
  [job scheduler](02-systems/job-scheduler/README.md)). A product-driven optimization — mention it; often deferred.

→ Next: **[Trade-offs](02-systems/notification-system/tradeoffs.md)**
