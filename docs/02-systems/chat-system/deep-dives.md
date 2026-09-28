# Chat / Messaging System — Deep Dives

The interview lives here: **routing a message to the right socket**, **ordering & dedup**, **receipts**, **presence**,
**group fan-out**, and **multi-device sync** — plus **E2EE** and **media** as named seams.

---

## 1. Connection routing

The core stateful-system problem: the sender's socket is on gateway A, the recipient's is on gateway B. **How does a
message on A reach the socket on B?**

- **`user → gateway` registry.** On connect, a gateway writes `user_id/device_id → gateway_id` to a shared store
  (Redis). To deliver, the message service looks up the recipient's gateway(s) and forwards.
- **Two delivery mechanisms:**
  - **Direct forward:** message service calls the owning gateway directly (RPC) using the registry. Simple; the
    registry is on the hot path.
  - **Pub-sub bus:** each gateway **subscribes to the users it currently holds**; the message service **publishes** to
    the recipient's channel; the owning gateway receives and pushes over the socket. Decouples sender from recipient
    location and scales fan-out better. This is the usual choice at scale.
- **Registry is ephemeral + TTL'd.** It's rebuilt on reconnect and kept fresh by heartbeats — **never the durable
  source of truth**. A stale entry (points to a dead gateway) just means delivery fails → fall back to offline/push +
  reconnect sync. The *message* is already persisted, so nothing is lost.
- **Sticky sessions**: a reconnecting client should land on a gateway predictably (consistent hashing / LB affinity)
  so its subscriptions and state are cheap to re-establish. See
  [real-time patterns](../../01-patterns/websockets-realtime.md#fan-out-routing-getting-a-message-to-the-right-server).

> **Interview line:** *"Each gateway registers the users it holds in a Redis map and subscribes to those users on a
> pub-sub bus. To deliver, I publish to the recipient's channel and their gateway pushes it down the socket. The
> registry is ephemeral and TTL'd — if it's stale, delivery just falls back to a push notification, because the
> message is already persisted."*

---

## 2. Ordering & deduplication

Two guarantees users feel immediately: **messages appear in the order sent**, and **a flaky network never creates
duplicates or holes.**

- **Per-conversation sequence number.** When the message service persists a message, it assigns the **next monotonic
  `seq` for that conversation**. This gives:
  - **Ordering:** clients render by `seq`, not by arrival time (network reorders; `seq` doesn't).
  - **Gap detection:** a client that has seq 1023 then receives 1025 **knows** it's missing 1024 → it requests the gap.
  - **A sync cursor:** "give me everything after seq X" is the reconnect catch-up (see #6).
- **Why per-conversation, not global?** Global ordering across all conversations needs a single sequencer bottleneck
  and nobody needs it — you only need order *within* a conversation. Per-conversation lets each conversation's
  sequencer scale independently (partitioned by `conversation_id`).
- **Assigning seq safely.** The seq source must be monotonic per conversation under concurrent sends: options are a
  per-conversation counter (atomic increment in the store / a per-partition sequencer), or the store's own clustering
  on an assigned value. The write is the serialization point — two simultaneous sends get 1024 and 1025, never both 1024.
- **Dedup via client-generated message id.** The client stamps each message with a **`clientMsgId` (UUID)** before
  sending. On a flaky reconnect the client may resend; the server checks `clientMsgId` and, if already written, returns
  the **original** `ack` (same seq) instead of writing again. This is [idempotency](../../01-patterns/idempotency.md)
  applied to chat → **at-least-once transport + dedup = the *effect* of exactly-once.**

> **Interview line:** *"Ordering and dedup both hang off two ids: a server-assigned per-conversation sequence number
> for order and gap detection, and a client-generated message id for idempotent retries. At-least-once delivery plus
> dedup on the client id gives me exactly-once *effect* without the impossible exactly-once *transport*."*

---

## 3. Delivery & read receipts

The sender wants **sent → delivered → read**. Each is a distinct fact with a distinct source.

- **Sent (✓)** — the server **ack'd** (persisted + assigned seq). Sender-side only.
- **Delivered (✓✓)** — the **recipient's device received** the message and sends back a `receipt{delivered, upTo:seq}`.
- **Read (✓✓ blue)** — the recipient **viewed** the conversation → `receipt{read, upTo:seq}`.
- **Cumulative, not per-message.** Receipts carry `upTo: seq` ("I have/read everything through 1024"), so N messages
  collapse into one receipt and receipts are **idempotent and self-healing** (a lost receipt is corrected by the next).
- **Receipts are a softer lane.** They're higher-volume than messages (delivered + read per message) but tolerate
  batching and slight delay → don't put them on the message write critical path; batch and debounce.
- **Group receipts are their own fan-out.** "Read by 40 of 100" means tracking read cursors per member — expensive in
  large groups, so many products **cap or drop per-member read receipts** beyond a group-size threshold.

> **Interview line:** *"Receipts are cumulative — the client acks 'I've read up to seq N' — so they're idempotent and
> collapse many messages into one signal. They ride a softer, batchable lane off the message write path, and in large
> groups I cap per-member read receipts because that's a fan-out in its own right."*

---

## 4. Presence & typing

"Online", "last seen", "typing…" — **high-frequency, ephemeral, best-effort.** Never let presence cost more than
messaging.

- **Source = the connection.** A user is "online" iff a gateway holds a live socket for them (heartbeat-fresh).
  Presence is derived from the same `user_connections` registry + TTL — no separate durable store.
- **TTL + heartbeat**: presence entries expire; a missed heartbeat → the socket is reaped → presence flips to offline
  automatically. **Self-healing**, no explicit "I'm leaving" needed.
- **Typing** is transient and **never persisted** — a `typing{start/stop}` frame fanned to the conversation's other
  online members, auto-expiring after a few seconds. Safe to drop under load.
- **Presence fan-out is the cost.** Broadcasting "u_7 came online" to all their contacts is bursty (morning login
  spikes). Mitigate with **debouncing/batching**, **subscribe-on-demand** (only compute presence for contacts the
  client is actually looking at), and accepting **staleness** (a few seconds late is fine).
- **Best-effort by design.** Presence is explicitly **eventually consistent** — trying to make it strongly consistent
  would cost more than the messaging system itself for near-zero product value.

> **Interview line:** *"Presence is derived from live connections with a TTL, so it's self-healing and never durably
> stored. Typing is a transient fan-out that's safe to drop. I debounce and compute presence on-demand for visible
> contacts, and I accept a few seconds of staleness — best-effort is a feature, not a compromise, here."*

---

## 5. Group fan-out

"How big can a group get?" is the highest-leverage question because **group size changes the delivery regime.**

- **Small groups (≤ ~hundreds): fan-out inline.** One `seq` assigned, persisted once; deliver to each member's live
  gateway(s), enqueue push for offline members. Cheap.
- **Large groups (thousands–10k+): fan-out via workers + queue.** Don't block the send path fanning out to 10k
  members; hand the persisted message to a **dedicated fan-out worker pool** fed by a
  [queue](../../01-patterns/queues-workers.md). The queue absorbs the burst; workers pace delivery.
- **Broadcast channels (100k–millions): flip to pull/feed.** Per-recipient push doesn't scale to millions per message.
  Store the message once and let clients **pull** on open (like a [news feed](../notification-system/README.md)), or a
  hybrid: push a lightweight "new message" nudge, client pulls the content. **Fan-out-on-write vs on-read** is the
  trade — see [trade-offs](tradeoffs.md#3-group-delivery-fan-out-on-write-vs-fan-out-on-read).
- **Hot partition.** A giant, chatty group concentrates writes on one `conversation_id` partition → a hotspot. Mitigate
  with fan-out workers, write batching, and (extreme) sub-partitioning a mega-group's message stream.

> **Interview line:** *"Small groups fan out inline; large groups go through a fan-out worker pool behind a queue so
> the send path never blocks; and a 100k+ broadcast channel I'd flip from push to pull — store once, clients pull on
> open — because per-recipient push to millions per message is the wrong regime."*

---

## 6. Multi-device sync

One account on phone + web + desktop, all converging on the same history and read state.

- **`user → set of connections`.** The routing registry maps a user to **all** their devices' gateways; every message
  is delivered to **every** online device.
- **Per-device delivery + a sync cursor.** Each device tracks its **last-seen `seq` per conversation**. On reconnect it
  sends `sync{cursors}`; the server replays everything after each cursor from the store. A device offline for a day
  catches up cleanly — **the store is truth, the socket is just the fast path.**
- **Read state converges via per-device cursors.** Each device reports `read_up_to`; the **account's** read position is
  the **max** across devices (read on the phone ⇒ don't re-badge on the laptop). Cursors are commutative → convergence
  without coordination.
- **Sending from one device, seeing it on your others.** Your own sent message is also delivered back to your *other*
  devices (echo) so all your screens match.
- **New device onboarding.** A freshly-installed device syncs from cursor 0 (or a bounded recent window) per
  conversation — bounded by retention/pagination, not a full firehose.

> **Interview line:** *"Multi-device is `user → many connections` plus a per-device, per-conversation sequence cursor.
> Every device gets every message, and each reports how far it's read; the account's read state is the max across
> devices. On reconnect a device just asks for everything after its cursor — the durable store makes catch-up trivial."*

---

## 7. End-to-end encryption (a mode, and what it removes)

- **What it is:** messages encrypted on the sender's device, decryptable only by recipients' devices (Signal protocol /
  Double Ratchet). The server routes **ciphertext** and never holds the keys.
- **What it *removes* architecturally:** **no server-side search**, **no server-side fan-out of plaintext** (the
  server can still route ciphertext, but can't transform/aggregate content), **no server-side content moderation**, and
  **per-device key management** (each device has keys; group messaging becomes N pairwise encryptions or sender-keys).
- **What still works:** sequence numbers, receipts, routing, presence, and offline push (push shows "New message", not
  content) — all operate on metadata/ciphertext.
- **Why ask early:** E2EE is a **first-class architectural constraint**, not a feature toggle — it reshapes search,
  moderation, and group fan-out. WhatsApp/Signal: yes; Slack/enterprise (needs search + compliance): usually no.

---

## 8. Media & attachments

- Media does **not** flow over the socket. Client requests a **presigned upload URL**, uploads to an **object store**,
  then sends a normal message whose body carries the **URL** (+ thumbnail, mime, size). Recipients fetch via **CDN**.
- Keeps the message store small (text + a URL), the socket lean, and egress off the messaging path (see
  [capacity](capacity.md#7-bandwidth-sanity-check)). Seam to a future file-storage system.

→ Next: **[Trade-offs](tradeoffs.md)**
