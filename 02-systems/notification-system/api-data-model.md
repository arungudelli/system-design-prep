# Notification System — API & Data Model

## APIs

### Send a notification (the main entry point, called by internal services)

```text
POST /v1/notifications
Idempotency-Key: <event-uuid>       # dedup: same event won't notify twice
{
  "recipientUserId": "u_123",       # or a segment/topic for broadcasts
  "category": "order_updates",      # for preference + opt-out checks
  "template": "order_shipped",
  "data": { "orderId": "A1", "eta": "Tue" },
  "channelsHint": ["push","email"], # optional; final channels resolved by preferences
  "priority": "transactional"       # transactional | bulk
}
→ 202 Accepted { "notificationId": "n_789" }     # async — accepted, not yet delivered
```

- **`202 Accepted`** — delivery is async; we accept fast and process behind a queue.
- **Idempotency-Key** (the triggering event id) → dedup so a retried/duplicate event doesn't double-notify
  (see [idempotency](01-patterns/idempotency.md)).
- **`priority`** routes to transactional vs bulk **lanes** (OTP must beat a campaign).
- **`category`** drives per-category preference/opt-out checks.

### Broadcast / segment send

```text
POST /v1/broadcasts { "segment": "all_us_users", "template": "sale", "priority": "bulk" }
→ 202 { "broadcastId": "b_1" }      # fanned out asynchronously to millions
```

### Preferences & devices (user-facing / app SDK)

```text
GET/PUT /v1/users/{id}/preferences  # per-channel, per-category opt-in/out, quiet hours, timezone
POST    /v1/users/{id}/devices      # register a push token (APNs/FCM)
DELETE  /v1/users/{id}/devices/{t}  # unregister (also auto-pruned on provider "invalid token")
GET     /v1/notifications?userId=   # in-app feed / history (paginated)
```

### Delivery status (callbacks + reads)

```text
POST /v1/webhooks/provider/{name}   # providers call us with delivery/bounce/open events
GET  /v1/notifications/{id}/status   # queued | sent | delivered | failed | opened
```

---

## Data model

### Access patterns first

1. **On fan-out:** read a user's **preferences + device tokens** (per notification) → hot, cache it.
2. **Dedup check:** "already notified for this event/channel?" → point lookup on a dedup key.
3. **Write delivery status** per send → very high write volume, append-y.
4. **In-app feed / history:** list a user's notifications, newest first.
5. **Template render:** fetch template by id + locale.

### `user_preferences` (small, hot, cacheable)

```text
user_id (PK)
channels: { push: on, email: on, sms: off }
categories: { marketing: off, order_updates: on, security: on }   # security often not opt-out-able
quiet_hours: { start, end }, timezone
locale
```
- Read on **every** fan-out → **[cache](01-patterns/caching.md)** it. Store in a KV/relational store; small.

### `devices` (push tokens)

```text
device_id (PK) | user_id (idx) | platform (ios/android/web) | push_token | last_seen | valid
```
- One user → many devices. Prune tokens when a provider reports "invalid/unregistered".

### `notifications` (dedup + status + feed)

```text
notification_id (PK)
event_id / idempotency_key (idx)     # dedup
user_id (idx)                        # feed / history
category, template, data
channels_sent: [ {channel, provider, status, attempts, sent_at, delivered_at} ]
status, created_at
```
- Serves **dedup** (by event_id), the **in-app feed** (by user_id, time-ordered), and **status**.
- **High write volume + huge over time** → the size driver; **TTL/retention** and consider a cheap append/columnar
  store or object storage for old records (see [capacity](02-systems/notification-system/capacity.md#storage)).

### `templates`

```text
template_id + locale (PK) | channel | subject/body with placeholders | version
```
- Tiny; render `data` into the template at send time. Version them for safe changes.

---

## SQL or NoSQL? (per store)

- **Preferences + devices → KV/relational, cached.** Small, hot, point-lookup by user — cache-friendly either way.
- **Notifications/status → NoSQL (wide-column/KV) or partitioned relational**, sharded by `user_id`. It's write-heavy,
  huge, and queried by user or event id — the KV/wide-column sweet spot; TTL old rows.
- **Templates → small relational/KV**, cached.
- **Feed reads (list by user, newest first)** → partition by `user_id`, sort by time — a wide-column store (e.g.
  Cassandra) fits naturally.

> **Interview line:** *"Preferences and device tokens are small and read on every fan-out, so I cache them.
> Delivery status/history is write-heavy and enormous, so it goes in a wide-column store sharded by user with a TTL
> — not the hot path DB. Templates are tiny and cached. As always, per access pattern, not by popularity."*

---

## The channel/provider abstraction (key interface)

```text
interface Channel {
  send(rendered, recipientAddress, idempotencyKey) -> {status, providerMessageId} | error
}
adapters: EmailChannel(SES|SendGrid), SmsChannel(Twilio|SNS), PushChannel(APNs|FCM), InAppChannel(DB write)
```
- A **uniform interface** with per-provider adapters lets you **fail over** between providers (SES → SendGrid) and
  isolate each provider's quirks, auth, and rate limits behind one seam. This is the core of channel resilience —
  detailed in [Deep Dives](02-systems/notification-system/deep-dives.md#1-the-channelprovider-abstraction).

→ Next: **[Architecture](02-systems/notification-system/architecture.md)**
