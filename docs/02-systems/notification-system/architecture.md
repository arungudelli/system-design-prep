# Notification System — Architecture

## 9. Architecture

The shape is a **pipeline of queues and worker pools**, splitting on channel and priority.

```text
 Internal services / events
            │  POST /notifications  (202 Accepted)
            ▼
      [ Ingestion API ]  validate · dedup (event_id) · resolve recipients · persist notification
            │  enqueue
            ▼
      [ Intake Queue ]
            │
            ▼
      [ Fan-out / Preference workers ]
          - expand recipient(s) → per-user
          - load preferences + devices (cached)  → drop opted-out / quiet-hours
          - render template (per locale)
          - choose channels; emit one message per (user, channel)
            │
   route by CHANNEL + PRIORITY
            ├───────────────┬───────────────┬───────────────┐
            ▼               ▼               ▼               ▼
     [push queue]     [email queue]    [sms queue]     [in-app queue]     (×2: transactional vs bulk lanes)
            │               │               │               │
     [push workers]   [email workers]  [sms workers]   [in-app writer]
       │  rate-limited to provider quota; retry/backoff; circuit breaker
       ▼
   Provider adapters → APNs/FCM · SES/SendGrid · Twilio/SNS
       │  delivery/bounce webhooks
       ▼
   [ Delivery tracking ] → status store (+ prune invalid device tokens)
```

**Every stage justified:**
- **Ingestion API** — accepts requests fast (`202`), validates, **dedups by event_id**, persists the notification,
  enqueues. Producers never block on downstream providers.
- **Fan-out workers** — the amplification stage: expand a request/broadcast into per-user, per-channel messages,
  applying **preferences + quiet hours + opt-out** and **template rendering**. This is where 1 event → millions.
- **Per-channel + per-priority queues** — email/SMS/push have different throughput, cost, and provider limits, and
  transactional must not wait behind bulk → **separate lanes**.
- **Channel workers** — pull from their queue, call the **provider adapter**, retry/backoff, respect provider rate
  limits, circuit-break a failing provider.
- **Delivery tracking** — consumes provider webhooks (delivered/bounced/opened), updates status, prunes dead tokens.

---

## 10. Request / data flow

### Transactional send (e.g. OTP) — the low-latency path
```text
1. Auth service → POST /notifications {priority: transactional, template: otp}
2. Ingestion: dedup(event_id) → persist → enqueue to TRANSACTIONAL intake
3. Fan-out: load prefs (OTP/security usually NOT opt-out-able) + device/phone → render
4. Route to the channel's TRANSACTIONAL queue (separate from bulk)
5. Channel worker → provider.send() with idempotency key → status=sent
6. Provider webhook → status=delivered
```
Transactional lanes are kept short and drained fast → OTP arrives in seconds even during a campaign.

### Broadcast (marketing) — the high-throughput path
```text
1. POST /broadcasts {segment, priority: bulk}
2. Fan-out expands the segment (streamed in batches — don't load 50M users at once)
3. Per user: check prefs/opt-out/quiet-hours → drop or enqueue to BULK channel queue
4. Bulk channel workers drain at provider-rate-limited pace (paced over minutes/hours)
```
The bulk lane is rate-limited + paced so it never starves transactional traffic or trips provider throttles.

### Failure path
```text
Provider 5xx/timeout → retry w/ backoff+jitter → after N → DLQ + alert
Provider 429 (throttle) → slow the lane (respect rate limit), requeue
Invalid device token → drop + prune token from devices
```

---

## 11. Bottlenecks (ranked)

| Rank | Bottleneck | Why | Detect via | Fix |
|---|---|---|---|---|
| 1 | **Fan-out amplification** | 1 event → millions of sends, bursty | intake→channel lag, queue depth | stream fan-out in batches; buffer in queues; autoscale fan-out workers |
| 2 | **Provider rate limits** | external quota per channel | 429s, provider throttle | per-channel **rate limiter**; pace campaigns; multi-provider failover |
| 3 | **Priority inversion** | bulk blocks transactional | OTP latency during campaigns | **separate transactional vs bulk lanes** |
| 4 | **Preference/device reads** | read on every fan-out | prefs store QPS | **cache** prefs + tokens |
| 5 | **Delivery-status writes** | ~250 GB/day, high write rate | write latency, storage growth | append store + TTL; batch writes |

The two signature problems are **fan-out amplification** and **not letting bulk drown transactional** — call these out.

---

## 12. Scaling each bottleneck

- **Fan-out** → **stream/batch** the recipient expansion (never `SELECT 50M users` into memory); autoscale fan-out
  workers on intake queue depth; the queues buffer the burst.
- **Provider limits** → a **[rate limiter](../../01-patterns/rate-limiting.md)** per channel matched to the provider quota;
  **pace** big campaigns to fit the quota over time; **multi-provider** so you can spread/fail over.
- **Priority** → physically **separate queues** for transactional vs bulk per channel; give transactional workers
  priority/dedicated capacity so an OTP never queues behind a campaign.
- **Preferences/devices** → **cache** (read on every send); invalidate on preference change.
- **Status writes** → batch, append-only store, TTL; don't write status to the hot path DB.

---

## 20. Evolution with scale

| Stage | Shape | What forced the jump |
|---|---|---|
| **1 · Small** | send synchronously in the request via one provider SDK | — |
| **2 · Moderate** | ingestion API + queue + workers + provider adapters + retries/DLQ | provider latency/failures blocking callers; need reliability |
| **3 · Large** | fan-out stage + **per-channel + per-priority queues** + preference cache + rate limiting | broadcasts amplify; OTPs stuck behind bulk; provider throttling |
| **4 · Extreme** | multi-provider failover, delivery-tracking pipeline, digests/aggregation, multi-region | provider outages; status-write volume; notification fatigue; global users |

> Transitions are driven by **fan-out bursts, provider unreliability, and priority isolation** — not raw steady QPS.

→ Next: **[Deep Dives](deep-dives.md)**
