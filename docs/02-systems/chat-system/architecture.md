# Chat / Messaging System — Architecture

## 9. Architecture

Two planes: a **real-time plane** (stateful gateways holding live sockets + a routing layer) and a **storage plane**
(the durable message store, the source of truth). The socket is the fast path; **the store is truth.**

```text
   Clients (phone / web / desktop)  ── persistent WebSocket ──┐
                                                              ▼
                                         [ Connection Gateways ]   (stateful, sticky, ~1k boxes)
                                          - hold live sockets, heartbeat/reap
                                          - register user→gateway in the routing store
                                          - terminate TLS, auth on connect
                                              │  send frame
                                              ▼
                                         [ Message Service ]
                                          - dedup on client_msg_id
                                          - assign per-conversation SEQ  (sequencer)
                                          - persist to message store  (durable = "accepted")
                                          - resolve recipients (members)
                                              │
                       ┌──────────────────────┼───────────────────────┐
                       ▼                       ▼                        ▼
             [ Routing / Pub-Sub ]     [ Message Store ]        [ Offline detector ]
              user→gateway registry     wide-column,             recipient has no live
              (Redis) + fan-out bus     partition=conv_id,       connection?
                       │                cluster=seq                     │
      route to each recipient's gateway(s)                              ▼
                       ▼                                        [ Notification System ]
             [ recipient Gateways ] ── push over socket ──►      (APNs/FCM push)
                       │                                                │
                  recipient clients ── receipt (delivered/read) ──►  back through the same path
```

**Every stage justified:**
- **Connection gateways** — the dominant, **stateful** tier. Hold hundreds of millions of live sockets, do
  heartbeat/reaping, auth-on-connect, and register `user→gateway` so others can find the socket.
  [Sticky sessions](../../01-patterns/websockets-realtime.md#sticky-sessions) route a reconnecting client back to a
  gateway; the socket state can't live behind a stateless LB.
- **Message service** — the write brain: **dedup** (client_msg_id), **assign the per-conversation sequence number**,
  **persist** (persist = the message is now safe, even if delivery fails), and resolve the recipient set from
  `conversation_members`.
- **Routing / pub-sub** — given recipients, find the gateway(s) holding their sockets via the `user→gateway` registry
  and deliver — directly or over an internal **pub-sub bus** (subscribe each gateway to the users it holds). This is
  the "route to the one server with the recipient's socket" problem (see [routing](deep-dives.md#1-connection-routing)).
- **Message store** — durable truth; wide-column, partitioned by `conversation_id`, clustered by `seq`.
- **Offline path** — if a recipient has no live connection, the message is *already persisted*; enqueue a
  **[push notification](../notification-system/README.md)**. They sync via cursor on reconnect.

---

## 10. Request / data flow

### Send, both parties online (the happy path)
```text
1. Sender → its gateway:  send{conversationId, clientMsgId, body}
2. Gateway → Message Service
3. Message Service: dedup(clientMsgId) → assign seq → PERSIST to store  (now durable)
4. Message Service → Routing: look up recipient(s) → gateway(s)
5. Recipient gateway → push message{seq} over recipient's socket(s)   (all their devices)
6. Gateway → sender:  ack{clientMsgId, serverMsgId, seq}   → sender shows "sent"
7. Recipient client → receipt{upTo: seq, delivered} → routed back to sender → "delivered"
8. Recipient opens chat → receipt{upTo: seq, read} → "read"
```
Note **persist before deliver** (step 3 before 5): a message that's been ack'd is guaranteed recoverable even if the
recipient's gateway dies mid-push.

### Send, recipient offline
```text
1–3. same: dedup → seq → persist  (message is safe)
4.   Routing finds NO live connection for the recipient
5.   Enqueue a push notification (via the Notification System)
6.   Sender gets ack{seq} → "sent" (not delivered)
7.   Recipient reconnects → sync{cursors} → server replays messages after their last seq → "delivered"
```

### Group send (fan-out)
```text
1–3. dedup → assign ONE seq for the conversation → persist ONCE
4.   Resolve members (conversation_members) → for each online member, route to their gateway(s);
     for each offline member, enqueue push
5.   Large group / broadcast → hand to a dedicated fan-out worker pool + queue (don't fan out inline)
```
One write, N deliveries — the amplification lives here (see [group fan-out](deep-dives.md#5-group-fan-out)).

### Failure path
```text
Recipient gateway down mid-deliver → message already persisted → delivered on reconnect via cursor. Never lost.
Message store write fails → do NOT ack → client retries same clientMsgId → idempotent (no dup).
Routing registry stale (points to a dead gateway) → deliver fails → fall back to offline/push + reconnect sync.
```

---

## 11. Bottlenecks (ranked)

| Rank | Bottleneck | Why | Detect via | Fix |
|---|---|---|---|---|
| 1 | **Connection fleet** | ~200M stateful sockets; RAM/conn bound | conns/box, gateway RAM, reconnect storms | horizontal gateway scaling; lean bytes/conn; shed/spread on reconnect |
| 2 | **Routing / fan-out** | every message → registry lookup + N deliveries | delivery lag, pub-sub throughput | pub-sub bus; dedicated fan-out workers for big groups |
| 3 | **Hot conversation / partition** | a giant group or celebrity broadcast concentrates writes+fan-out | one partition's write latency, queue depth | fan-out workers + queue; cap/shard huge groups; treat broadcast as a feed |
| 4 | **Message store writes** | ~2.5M msg/s peak + receipts | write latency, compaction | partition by conv_id; scale wide-column; receipts as softer lane |
| 5 | **Reconnect thundering herd** | a gateway/region drop → millions reconnect at once | connect rate spike | backoff+jitter on clients; gateway accept rate-limiting |

The signature problems: **holding the connections** and **routing/fan-out a message to the right socket(s)** — call
these out first.

---

## 12. Scaling each bottleneck

- **Connection fleet** → scale gateways **horizontally**; sized by concurrent connections + RAM/connection. Drive
  down bytes/connection. Use consistent-hashing / sticky routing so reconnects land predictably.
- **Routing/fan-out** → an internal **pub-sub bus**: each gateway subscribes to the users it holds; the message
  service publishes to the recipient's channel and the owning gateway delivers. Big groups go through **dedicated
  fan-out workers + a queue** rather than fanning out on the send path.
- **Hot conversation** → don't fan out a 100k-member broadcast inline; queue it and let a worker pool pace delivery;
  for celebrity broadcast, treat it like a **feed/pull** problem, not per-recipient push.
- **Message store** → partition by `conversation_id` spreads writes; scale the wide-column cluster; keep **receipts
  and presence off the message write path** (softer, batchable lanes).
- **Reconnect storms** → client **backoff + jitter**, server-side **accept rate limiting**, and spreading
  reconnects across the fleet so one failed gateway doesn't stampede its neighbors.

---

## 20. Evolution with scale

| Stage | Shape | What forced the jump |
|---|---|---|
| **1 · Single server** | one process holds all sockets in memory + a local DB; deliver by looking up the socket in a local map | runs out of RAM/CPU; one box = SPOF; can't hold enough connections |
| **2 · Gateways + shared store** | multiple stateful gateways + a shared message store + a **user→gateway registry** to route across boxes | more than one gateway → "which server has the recipient's socket?" |
| **3 · Pub-sub routing + fan-out workers + offline push (base target)** | internal **pub-sub bus** for cross-gateway delivery; dedicated **fan-out** for groups; **[notification](../notification-system/README.md)** for offline; wide-column store partitioned by conv_id | group amplification; offline users; routing at scale |
| **4 · Sharded/multi-region, per-conv sequencing, broadcast-as-feed, E2EE** | sharded connection tier, region-local gateways, a robust per-conversation sequencer, broadcast channels served pull-style, optional end-to-end encryption | hundreds of millions of connections, global latency, huge broadcast groups, privacy requirements |

> Transitions are driven by **connection count, cross-server routing, and fan-out amplification** — not steady send
> QPS. The move from Stop 1→2 (one box → "find the recipient's socket on another box") is the conceptual leap.

→ Next: **[Deep Dives](deep-dives.md)**
