# Chat / Messaging System — Interview Questions

> Answer each **out loud first**, then expand and self-critique against the
> [critique lens](../../00-framework/staff-level-thinking.md#how-your-answer-gets-graded-the-critique-lens).

---

## Core

**Q1. What's the overall shape of this system?**
<details markdown="1"><summary>Model answer</summary>

**Two planes.** A **real-time plane** — stateful WebSocket gateways holding hundreds of millions of live sockets, plus
a routing layer — and a **storage plane** — a durable wide-column message store that's the source of truth. Send path:
client → gateway → message service (**dedup on client id → assign per-conversation seq → persist**) → route to the
recipient's gateway via a `user→gateway` registry / pub-sub → push over the socket → ack back. Offline recipients get
a push notification and sync on reconnect. See [architecture](architecture.md).
</details>

**Q2. Why WebSockets, and what does holding the connections cost?**
<details markdown="1"><summary>Model answer</summary>

Chat is bidirectional and latency-sensitive → **WebSocket** (full-duplex, low per-message overhead), with long-poll as
a fallback. The cost is the **connection fleet**: chat is **connection-bound, not request-bound** — ~200M concurrent
sockets sized by **RAM/connection** (~2 TB just to hold idle sockets), plus ~6.6M heartbeat frames/sec before any real
message. You pay for who's *connected*, not who's *talking*. See [capacity](capacity.md#1-connection-fleet-the-dominant-cost-size-this-first).
</details>

**Q3. A message is sent from a socket on gateway A to a recipient whose socket is on gateway B. How does it get there?**
<details markdown="1"><summary>Model answer</summary>

Each gateway registers the users it holds in a shared **`user→gateway` registry** (Redis) and **subscribes to those
users on a pub-sub bus**. The message service persists the message, then **publishes to the recipient's channel**; the
owning gateway receives it and pushes it down the socket. The registry is **ephemeral + TTL'd** — if it's stale,
delivery falls back to a push notification because the message is already persisted. See [routing](deep-dives.md#1-connection-routing).
</details>

**Q4. How do you guarantee ordering and no duplicates over a flaky network?**
<details markdown="1"><summary>Model answer</summary>

Two ids. A server-assigned **per-conversation sequence number** gives ordering (clients render by seq, not arrival) and
**gap detection** (missing 1024 between 1023 and 1025 → request it). A **client-generated message id** makes retries
idempotent — a resend returns the original ack instead of writing a duplicate. **At-least-once transport + dedup =
exactly-once *effect*.** See [ordering & dedup](deep-dives.md#2-ordering--deduplication).
</details>

**Q5. Why per-conversation ordering and not global?**
<details markdown="1"><summary>Model answer</summary>

Users only perceive order **within** a conversation; cross-conversation total order is meaningless and would force a
single global sequencer bottleneck. Per-conversation seq **partitions** — each conversation's sequencer scales
independently (partitioned by `conversation_id`). See [trade-offs](tradeoffs.md#4-ordering-global-vs-per-conversation).
</details>

---

## Storage & delivery

**Q6. How do you store messages, and why that layout?**
<details markdown="1"><summary>Model answer</summary>

A **wide-column store partitioned by `conversation_id`, clustered by `seq`**. The dominant reads are "latest 50 in a
conversation" (open a chat) and "everything after seq X" (reconnect sync) — both **single-partition range scans** in
this layout. It's write-heavy and petabyte-scale, which is the wide-column sweet spot; media is kept out (URL only).
A giant group is a **hot partition**, handled with fan-out workers. See [data model](api-data-model.md#messages--the-source-of-truth-wide-column).
</details>

**Q7. Walk delivery + read receipts. What do they cost?**
<details markdown="1"><summary>Model answer</summary>

**sent** (server ack'd) → **delivered** (recipient device received, `receipt{delivered, upTo:seq}`) → **read**
(recipient viewed, `receipt{read, upTo:seq}`). Receipts are **cumulative** ("I've read through seq N") so they're
idempotent, self-healing, and collapse N messages into one. They're a **softer, batchable lane** off the message write
path. In **large groups**, per-member read receipts are their own fan-out → cap or drop them beyond a size threshold.
See [receipts](deep-dives.md#3-delivery--read-receipts).
</details>

**Q8. What happens when the recipient is offline?**
<details markdown="1"><summary>Model answer</summary>

The message is **persisted before any delivery attempt**, so it's already safe. Routing finds no live connection →
enqueue a **[push notification](../notification-system/README.md)**. The sender sees "sent." On reconnect the client
sends `sync{cursors}` and the server replays everything after its last seq → "delivered." Delayed, never dropped.
</details>

---

## Scale & fan-out

**Q9. How do groups change the design? What about a 100k-member broadcast channel?**
<details markdown="1"><summary>Model answer</summary>

Group size changes the **delivery regime**. Small groups **fan out on write** inline (one seq, persist once, deliver to
each member). Large groups (thousands+) fan out via a **dedicated worker pool + queue** so the send path never blocks.
A 100k+ **broadcast channel** flips to **fan-out on read / pull** — store once, clients pull on open (optionally a
lightweight push nudge) — because per-recipient push to millions per message is the wrong regime. "How big can a group
get?" is the highest-leverage clarifying question. See [group fan-out](deep-dives.md#5-group-fan-out).
</details>

**Q10. What's the first bottleneck as you scale?**
<details markdown="1"><summary>Model answer</summary>

The **connection fleet** (holding ~200M stateful sockets, RAM-bound) and **routing/fan-out** (every message → registry
lookup + N deliveries). Fix: scale gateways horizontally + drive down bytes/connection; a pub-sub bus + dedicated
fan-out workers for groups. Then hot conversations (queue + pace, or flip broadcast to pull) and reconnect storms
(backoff + jitter + accept rate-limiting). See [bottlenecks](architecture.md#11-bottlenecks-ranked).
</details>

**Q11. How does multi-device work — phone, web, and desktop on one account?**
<details markdown="1"><summary>Model answer</summary>

`user → set of connections`; every message is delivered to **every** online device, and your own sent messages echo to
your other devices. Each device keeps a **per-conversation seq cursor** and syncs "everything after my cursor" on
reconnect. Read state converges via **per-device `read_up_to` cursors** — the account's read position is the **max**
across devices, so reading on the phone clears the badge on the laptop. See [multi-device sync](deep-dives.md#6-multi-device-sync).
</details>

---

## Presence & reliability

**Q12. How do presence and "typing…" work, and how accurate are they?**
<details markdown="1"><summary>Model answer</summary>

Both are **ephemeral, best-effort, never persisted**. Presence is **derived from live connections + TTL** — a missed
heartbeat reaps the socket and flips presence to offline automatically (self-healing). Typing is a transient fan-out
that's safe to drop. Presence fan-out (notifying contacts) is bursty, so I **debounce**, compute **on-demand** for
visible contacts, and accept a few seconds of staleness. Strong-consistency presence would cost more than messaging for
near-zero value. See [presence](deep-dives.md#4-presence--typing).
</details>

**Q13. A gateway holding a million connections crashes. What happens?**
<details markdown="1"><summary>Model answer</summary>

All its sockets drop, but the gateway held **no source-of-truth state** — only sockets + an ephemeral registry.
Persisted messages are safe; undelivered ones become offline → push + reconnect sync. Clients reconnect with **backoff
+ jitter** and are spread across surviving gateways; the registry rebuilds from reconnects. The connection tier is
**disposable by design**. See [failure scenarios](failure-scenarios.md#a-connection-gateway-crashes).
</details>

**Q14. Why "persist before deliver," and what if the store write fails?**
<details markdown="1"><summary>Model answer</summary>

Persisting first makes the message recoverable even if delivery fails (gateway crash, offline recipient) — the store is
truth, the socket is a fast path. If the write fails, **don't ack**; the client retries with the same `clientMsgId`,
which is **idempotent** on eventual success. Better a spinner than a lost or duplicated message. See
[failure scenarios](failure-scenarios.md#message-store-write-fails--is-slow).
</details>

---

## Staff-level curveballs

**Q15. What changes if it must be end-to-end encrypted?**
<details markdown="1"><summary>Model answer</summary>

E2EE is a **first-class constraint**, not a toggle. The server routes **ciphertext** and holds no keys → **no
server-side search, no plaintext fan-out/transformation, no content moderation**, and **per-device key management**
(group = sender-keys or N pairwise). Sequence numbers, receipts, routing, presence, and offline push still work on
metadata/ciphertext (push shows "New message," not content). Ask up front — WhatsApp/Signal yes, Slack no. See
[E2EE](deep-dives.md#7-end-to-end-encryption-a-mode-and-what-it-removes).
</details>

**Q16. Where does this reuse patterns you've studied?**
<details markdown="1"><summary>Model answer</summary>

**WebSockets/real-time:** the connection tier, sticky sessions, heartbeats, reconnect/resume, fan-out routing.
**Queues + workers:** group fan-out and offline push. **Idempotency:** client-id dedup. **Sharding:** message store by
`conversation_id`, connection tier by user. **Caching:** the routing registry (hot). **Notifications:** offline
delivery. **Rate limiting:** reconnect accept limits + per-user send caps. It composes the whole pattern library around
a stateful connection tier.
</details>

**Q17. What changes at 10× and 100×?**
<details markdown="1"><summary>Model answer</summary>

**10×:** more gateways, a bigger pub-sub bus, harder hot-group handling. **100×:** a **sharded connection tier**,
**multi-region** region-local gateways (global latency + data residency), broadcast channels served **pull-style as a
feed**, a robust per-conversation **sequencer**, and possibly **E2EE**. The two-plane backbone holds; pressure is on
connection count, cross-region routing, and fan-out amplification — not steady send QPS. See [principal deep dive](principal-deep-dive.md).
</details>

**Q18. When would you redesign this system?**
<details markdown="1"><summary>Model answer</summary>

When assumptions break: **broadcast/creator channels** dominate (it becomes a **feed/pub-sub** system, not 1:1 chat);
**E2EE + compliance search** both required (fundamentally conflicting → rethink); **real-time collaboration** (docs/
presence-cursors) needs CRDTs/OT, not message append; or global **five-nines + data residency** forces multi-region
active-active. Absent those, the two-plane design holds.
</details>

---

## Self-scoring

Grade against the [15-point critique lens](../../00-framework/staff-level-thinking.md#how-your-answer-gets-graded-the-critique-lens):
did you split real-time vs storage planes, persist before deliver, use a per-conversation sequence number for
ordering/gap/sync, dedup on a client id for exactly-once effect, solve cross-gateway routing, pick a fan-out regime by
group size, keep presence/receipts best-effort, and make the connection tier disposable behind a durable store? Note
your weakest area and drill it.

← Back to **[README](README.md)** · Next: **[Cheatsheet](cheatsheet.md)**
