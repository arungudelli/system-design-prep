# Chat / Messaging System — Trade-offs

> Dominant requirements: **massive concurrent-connection scale**, **low-latency ordered delivery**, **no message
> loss**, and **multi-device sync** — with presence/receipts explicitly allowed to be softer. Judge every trade
> against those.

---

## 1. Transport: WebSocket vs SSE vs long-poll

- **Long-poll:** works everywhere, no special infra; but high overhead, awkward for bidirectional, and not truly
  instant.
- **SSE (server-sent events):** simple server→client streaming over HTTP; but **one-directional** (needs a separate
  channel for sends) and no binary.
- **WebSocket:** full-duplex, low per-message overhead, binary-capable — the natural fit for chat's bidirectional,
  high-frequency traffic.

**Choose:** **WebSocket** for the primary transport, with **long-poll as a fallback** for restrictive networks. Chat
is inherently bidirectional and latency-sensitive. See the
[transport staircase](../../01-patterns/websockets-realtime.md).

---

## 2. Delivery: push over the socket vs client pull

- **Push:** server pushes to the recipient's live socket → instant. But requires holding connections and solving
  routing/fan-out.
- **Pull:** clients poll for new messages → simple, stateless, but not instant and wasteful at idle.

**Choose:** **push for online 1:1 and small groups** (the whole point is instant), **pull for huge broadcast
channels** (millions of recipients per message make push the wrong regime — see #3), and **push-notification + pull**
for offline recipients.

---

## 3. Group delivery: fan-out on write vs fan-out on read

- **Fan-out on write (push to each member's inbox/socket):** delivery is instant and reads are cheap; but a
  10k-member group means 10k deliveries per message, and a broadcast channel is catastrophic.
- **Fan-out on read (store once, members pull on open):** one write regardless of group size; but adds read latency
  and load on open, and "instant" becomes "on next fetch."

**Choose:** **write-fan-out for small/medium groups** (instant, manageable amplification) and **read-fan-out for very
large groups / broadcast channels** (store once, pull on open, optionally a lightweight push nudge). This is the same
**push-vs-pull / celebrity-problem** trade as feeds. See [group fan-out](deep-dives.md#5-group-fan-out).

---

## 4. Ordering: global vs per-conversation

- **Global ordering:** a single total order across all messages — conceptually clean, but needs one sequencer
  bottleneck and **nobody needs it**.
- **Per-conversation ordering:** a monotonic seq per conversation — all users actually require, and it **partitions**
  (each conversation's sequencer scales independently).

**Choose:** **per-conversation ordering** via a per-conversation sequence number. Order within a chat is what users
perceive; cross-chat order is meaningless. See [ordering & dedup](deep-dives.md#2-ordering--deduplication).

---

## 5. Delivery guarantee: at-least-once + dedup vs at-most-once

- **At-most-once:** never duplicates, but can **drop** — unacceptable; losing a message is the cardinal sin.
- **At-least-once + dedup:** never drops; a retry is made idempotent by the **client message id** → the *effect* of
  exactly-once.

**Choose:** **at-least-once + dedup.** Exactly-once *transport* is impossible over flaky networks; exactly-once
*effect* via idempotent client ids is the achievable, correct target.

---

## 6. Message store: single shared store vs per-user inbox copies

- **Single shared conversation store (fan-out on read):** one copy per message, partitioned by `conversation_id`;
  "latest N in a conversation" is a single-partition scan. Cheaper storage; the natural model.
- **Per-user inbox (fan-out on write, a copy per recipient):** each user has their own materialized message list;
  reads are dead simple but storage multiplies by group size and writes amplify.

**Choose:** **shared conversation store** as the source of truth (partition by `conversation_id`, cluster by `seq`);
add a **per-user inbox/index only where write-fan-out is chosen** (small/medium groups) as a delivery convenience, not
the truth. See [data model](api-data-model.md#data-model).

---

## 7. Connection tier: stateful sticky gateways vs stateless

- **Stateless:** trivial to scale and load-balance; but a WebSocket **is** state (a live socket bound to a process) —
  you can't make holding a connection stateless.
- **Stateful sticky gateways:** accept the state; route reconnects back predictably; register `user→gateway` so others
  can find the socket.

**Choose:** **stateful gateways with sticky routing** — inherent to persistent connections. The design work is making
the state cheap (lean bytes/conn) and recoverable (ephemeral registry + durable store behind it).

---

## 8. Presence: strong vs best-effort (eventually consistent)

- **Strong/consistent presence:** always-accurate online status; but enormously expensive at hundreds of millions of
  flapping connections, for near-zero product value.
- **Best-effort presence:** TTL + heartbeat-derived, debounced, slightly stale — cheap and self-healing.

**Choose:** **best-effort, eventually consistent presence** (and typing). Explicitly softer than messaging — this is a
deliberate correctness/cost trade, not laziness. See [presence](deep-dives.md#4-presence--typing).

---

## 9. Sequencing source: centralized sequencer vs store-assigned

- **Central sequencer service:** clean monotonic ids; but another tier to run and a potential bottleneck/SPOF per
  conversation.
- **Store-assigned (atomic increment / partition-local):** the write itself is the serialization point; fewer moving
  parts.

**Choose:** **store/partition-assigned per-conversation seq** in most designs (the write is already the
serialization point); a dedicated sequencer only if the store can't provide monotonicity cheaply.

---

## 10. Encryption: E2EE vs server-readable

- **E2EE:** maximum privacy; but **removes server-side search, plaintext fan-out, and moderation**, and adds
  per-device key management.
- **Server-readable (TLS in transit + at rest):** enables search, moderation, and richer server features; the server
  can read content.

**Choose:** **product-driven.** Consumer-privacy apps (WhatsApp/Signal) → E2EE; enterprise/collaboration needing
search + compliance (Slack) → server-readable. Ask up front — it reshapes the design. See
[E2EE](deep-dives.md#7-end-to-end-encryption-a-mode-and-what-it-removes).

---

## The one-paragraph summary (say this)

> *"It's a two-plane system: stateful WebSocket gateways holding the live connections, and a durable wide-column store
> that's the source of truth. On send I dedup on a client-generated message id, assign a per-conversation sequence
> number, persist, then route to the recipient's gateway via a `user→gateway` registry and a pub-sub bus — persist
> before deliver, so nothing is lost if a gateway dies. Delivery is at-least-once with dedup for exactly-once effect,
> ordered per conversation by the sequence number, which doubles as the reconnect sync cursor. Small groups fan out on
> write, huge broadcast channels flip to pull. Presence and typing are best-effort, TTL-derived from connections.
> Multi-device is `user → many connections` with per-device read cursors that converge on the max. Offline recipients
> get a push notification and sync on reconnect."*

→ Next: **[Failure Scenarios](failure-scenarios.md)**
