# Chat / Messaging System

> Design a real-time messaging system (WhatsApp / Messenger / Slack) — **1:1 and group chat**, delivered in
> **milliseconds**, with **presence** ("online", "typing…"), **delivery/read receipts**, **history** that syncs
> across a user's **multiple devices**, and **push notifications** when a recipient is offline — at a scale of
> **hundreds of millions of concurrent connections** and **billions of messages a day**.

The naive version ("POST a message, the other person polls for it") misses the entire interview. Chat is the
canonical **stateful-connection** system: the hard parts are holding hundreds of millions of live
[WebSocket connections](../../01-patterns/websockets-realtime.md), **routing a message to the one server that
holds the recipient's socket**, guaranteeing **ordered, exactly-once-*effect* delivery** over flaky networks,
**fanning out** group messages, keeping **multiple devices in sync**, and falling back to
[push notifications](../notification-system/README.md) when the recipient is gone.

---

## Why it's a great system to study

- **It forces the real-time tier front and center** — this is where [polling → WebSockets](../../01-patterns/websockets-realtime.md),
  sticky sessions, heartbeats, reconnection, and fan-out routing stop being theory.
- **The message store is a genuine data-modeling problem** — "get the last 50 messages in this conversation,
  ordered, fast" at billions/day is a wide-column / partition-key decision, not a `SELECT * FROM messages`.
- **Delivery semantics are subtle** — ordering, dedup, sent/delivered/read receipts, and **multi-device sync** are
  where staff-level rigor shows: TCP delivery ≠ application delivery.
- **It composes everything** — connections ([realtime](../../01-patterns/websockets-realtime.md)), durable
  fan-out ([queues](../../01-patterns/queues-workers.md)), dedupe ([idempotency](../../01-patterns/idempotency.md)),
  and offline delivery ([notifications](../notification-system/README.md)).

---

## Read in this order

1. **[Requirements](requirements.md)** — problem, clarifying questions, functional + NFRs
2. **Capacity** *(coming)* — connections, messages/sec, storage, fan-out
3. **API & Data Model** *(coming)* — WebSocket protocol, message/conversation schema, sequence numbers
4. **Architecture** *(coming)* — connection tier → routing → message service → store → offline push
5. **Deep Dives** *(coming)* — routing, ordering & dedup, receipts, presence, group fan-out, multi-device sync
6. **Trade-offs** *(coming)* — the decisions, both sides
7. **Failure Scenarios** *(coming)* — connection drops, split brain, dup/lost messages, hot groups
8. **Interview Questions** *(coming)* — attempt first, then reveal
9. **Cheatsheet** *(coming)* — 1-page revision
10. **★ Principal Deep Dive** *(coming)* — the **staged stops** (single server → global) + max-scale
    variant (sharded connection tier, per-conversation sequencing, multi-region, E2E encryption) with a
    reconciliation table of when to adopt/remove each piece

---

## The 30-second version (know this cold)

- **Two planes.** A **real-time plane** (stateful WebSocket gateways holding live connections) and a **storage
  plane** (durable message history). The socket is the fast path; **the store is the source of truth.**
- **Send path:** client → its **gateway** → **persist message** (assign a per-conversation **sequence number**) →
  **route** to the recipient's gateway(s) via a `user → gateway` registry / pub-sub → push over their socket →
  **ACK** back to sender ("delivered").
- **Recipient offline?** Message is already persisted; enqueue a **[push notification](../notification-system/README.md)**.
  They sync missed messages (via the sequence cursor) on reconnect.
- **Ordering & dedup:** a **monotonic sequence number per conversation** gives ordering and gap detection;
  **client-generated message IDs** make retries idempotent → the *effect* of exactly-once.
- **Store:** wide-column (Cassandra/HBase-style), **partitioned by `conversation_id`**, **clustered by sequence/
  time** — reads are "latest N in a conversation," which this layout serves in one partition scan.
- **Group chat = fan-out:** write once, deliver to every member's connection(s); large groups amplify this.
- **Presence & typing:** ephemeral, heartbeat + TTL, **best-effort** — never stored durably, never strongly consistent.
- **Multi-device:** `user → set of connections`; every device gets the message and a per-device **read cursor** so
  read state converges.
- **Bottlenecks:** the **connection fleet** (sized by concurrent connections, not RPS), **routing/fan-out**, and
  **hot conversations** (a giant group or a celebrity broadcast).
