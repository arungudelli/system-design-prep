# Chat / Messaging System — Principal-Level Deep Dive (The Stops)

> The **"never be blind"** companion to the base Chat docs. The system as a **staircase of stops** — smallest
> defensible design first, each bigger stop with its **exact trigger** — ending in the **top-stop (max-scale /
> deliberately over-engineered)** architecture plus a **reconciliation table** for when each heavy component is overkill
> and should be **removed**.
>
> Read the base first: [Requirements](requirements.md) · [Architecture](architecture.md)
> · [Deep Dives](deep-dives.md) · [Trade-offs](tradeoffs.md).
>
> **Say this in an interview:** *"I'll start with a single server holding sockets in memory and grow it one bottleneck
> at a time. I know the sharded, multi-region, pull-for-broadcast, E2EE version — but each piece only earns its place
> when a specific ceiling forces it."*

---

## The stops (climb one lever at a time)

| Stop | Scale target | Shape | **Trigger that forces the NEXT stop** |
|---|---|---|---|
| **1 · Single server** | thousands of users | one process holds all WebSockets in an in-memory map; deliver by local lookup; one DB for history | runs out of RAM/CPU on one box; SPOF; can't hold enough connections |
| **2 · Gateways + shared store + registry** | ~1M connections | multiple **stateful gateways** + shared message store + a **`user→gateway` registry** to route across boxes; persist-before-deliver; per-conversation **seq** + **clientMsgId dedup** | "which gateway has the recipient's socket?" at scale; group amplification; offline users |
| **3 · Pub-sub routing + fan-out workers + offline push (base target)** | 100M+ connections | internal **pub-sub bus** for cross-gateway delivery; **dedicated fan-out** for groups; **[notification](../notification-system/README.md)** for offline; wide-column store partitioned by conv_id; best-effort presence | hundreds of millions of connections; global latency; huge broadcast groups; privacy/compliance |
| **4 · Sharded + multi-region + broadcast-as-feed + E2EE (top stop)** | 100M+ concurrent, global | **sharded connection tier**, **region-local gateways** + geo-routing, robust **per-conversation sequencer**, **broadcast channels served pull-style**, optional **E2EE**, per-tenant isolation | (ceiling; a creator-broadcast product becomes a feed system, not 1:1 chat) |

> Most interviews want **Stop 2–3**. Reach into Stop 4 only when the prompt says "global / hundreds of millions of
> connections / massive broadcast channels / end-to-end encrypted." Name the trigger each time.

---

## Top-stop architecture (max-scale)

```text
   Clients (multi-device) ── WebSocket (geo-routed to nearest region) ──┐
                                                                        ▼
                             [ Sharded Connection Gateways ]  region-local, sticky, consistent-hashed by user
                              - hold sockets, heartbeat/reap, subscribe to held users on the bus
                                        │ send
                                        ▼
                             [ Message Service ]  dedup(clientMsgId) · PER-CONVERSATION SEQUENCER · persist
                                        │
                ┌───────────────────────┼───────────────────────────┐
                ▼                        ▼                           ▼
        [ Pub-Sub Bus ]          [ Message Store ]            [ Fan-out Fleet ]
         (region-aware,           wide-column, sharded          - small/med groups: push per member
          user-channel)          partition=conv_id,             - BROADCAST channels: write once,
                │                 cluster=seq, tiered              clients PULL on open (feed model)
     route to recipient gateway(s)  (hot → cold/object store)          │
                ▼                                                       ▼
        recipient Gateways ── push ──► clients          offline members → [ Notification System ] (push)
                                                                        │
   Cross-region replication of the message store · geo-routing + failover · optional E2EE (server routes ciphertext)
   Per-tenant quotas + reconnect accept-limiting + presence (best-effort, region-local)
```

---

## Deep dives that only matter at the top stop

### Sharded connection tier (Stop 3 → 4 trigger: **connection count outgrows a flat fleet / routing hotspots**)
- **Why:** at hundreds of millions of sockets, a flat gateway pool + one registry becomes a routing and rebalancing
  problem; you need predictable placement and blast-radius isolation.
- **Design:** **consistent-hash users to gateway shards**; registry + pub-sub become shard/region-aware; reconnects land
  predictably on the owning shard; losing a shard drops only its slice.
- **Over-engineering watch:** *only when a single flat fleet + Redis registry actually strains.* Below that, flat
  gateways with sticky routing are simpler; **remove** the sharding layer.

### Multi-region gateways + geo-routing (trigger: **global users + latency / data residency / five-nines**)
- **Why:** a user in Asia shouldn't hold a socket in us-east; regional SPOF; some data must stay in-region (GDPR).
- **Design:** **region-local gateways** (connect to nearest), cross-region **message-store replication**, geo-routing
  with failover; the pub-sub bus routes cross-region only when sender and recipient are in different regions.
- **Cross-region ordering caveat:** per-conversation seq must still be assigned at a single authority per conversation
  (home region) to stay monotonic — cross-region sends round-trip to the conversation's home sequencer.
- **Over-engineering watch:** *only for global latency / residency / five-nines.* Single-region multi-AZ serves most;
  **remove** multi-region below that.

### Broadcast channels as a feed (trigger: **groups reach 100k–millions / creator channels**)
- **Why:** per-recipient push of one message to millions is the wrong regime (it's a fan-out bomb).
- **Design:** **flip to fan-out-on-read** — store the message once, clients **pull on open** (paged), optionally a
  lightweight push "nudge." This is the [feed / celebrity-problem](tradeoffs.md#3-group-delivery-fan-out-on-write-vs-fan-out-on-read)
  pattern; at that point a broadcast channel is closer to a **news feed** than to chat.
- **Over-engineering watch:** *only for very large channels.* Normal groups fan out on write and feel instant;
  **remove** the pull path for sub-thousand groups.

### Robust per-conversation sequencer (trigger: **concurrent-send correctness at high write rate / cross-region**)
- **Why:** monotonic per-conversation seq under heavy concurrent sends (and across regions) needs a clear serialization
  authority.
- **Design:** a **per-conversation sequencer** (partition-local atomic counter, or a dedicated sequencer sharded by
  conv_id) that is the single point that assigns seq; the write is the serialization point.
- **Over-engineering watch:** *only when store-native monotonicity isn't enough.* Most stores give you a
  partition-local monotonic value for free; **remove** the dedicated sequencer below that.

### End-to-end encryption (trigger: **privacy product / regulatory requirement**)
- **Why:** the server must not be able to read messages.
- **Design:** clients encrypt; server routes **ciphertext** and manages **per-device key distribution** (Signal /
  sender-keys for groups). Sequence, receipts, routing, presence, offline push all still work on metadata.
- **What it removes:** server-side **search, plaintext fan-out/transform, and moderation** — a first-class
  architectural constraint, not a toggle. See [E2EE](deep-dives.md#7-end-to-end-encryption-a-mode-and-what-it-removes).
- **Over-engineering watch:** *only when the product/regulator requires it.* Enterprise search + compliance often
  *forbid* E2EE; **remove** it when server-readability is a feature.

### Per-tenant isolation & quotas (trigger: **multi-tenant / noisy-neighbor**)
- **Why:** one org's mega-group or reconnect storm must not degrade another's messaging.
- **Design:** per-tenant [rate limits](../../01-patterns/rate-limiting.md), connection quotas, and (extreme) dedicated
  gateway pools for large tenants; fair scheduling on shared fan-out capacity.
- **Over-engineering watch:** *only for real multi-tenant contention.* Single-product/consumer scale doesn't need it;
  **remove** the isolation layer.

---

## Reconciliation table — what forces each heavy component (and when to remove it)

| Component | Base docs (Stop 2–3) | Top stop (Stop 4) | Trigger to ADOPT | When it's OVERKILL → remove |
|---|---|---|---|---|
| Connection tier | flat stateful gateways + sticky | consistent-hash **sharded** tier | flat fleet/registry strains; routing hotspots | flat + sticky is enough |
| Regions | single-region multi-AZ | multi-region gateways + geo-routing + replication | global latency / residency / five-nines | 99.99%, one geo |
| Big-group delivery | fan-out on write via workers | **broadcast = fan-out on read (pull)** | groups reach 100k–millions | sub-thousand groups |
| Sequencer | store/partition-assigned seq | dedicated per-conversation sequencer | high concurrent-send / cross-region monotonicity | store-native monotonic value suffices |
| Encryption | TLS in transit + at rest | **E2EE** (ciphertext routing, per-device keys) | privacy product / regulation | search + compliance need server-readable |
| Tenant isolation | shared fleet | per-tenant quotas + dedicated pools | multi-tenant noisy-neighbor | single/consumer product |
| Routing | Redis registry + direct/pub-sub | region-aware pub-sub bus | cross-gateway + cross-region delivery | one region, modest fleet |

> Lead with Stop 2–3 (stateful gateways + `user→gateway` registry + pub-sub routing + per-conversation seq + clientMsgId
> dedup + persist-before-deliver + offline push + best-effort presence). When pushed: *"for global, hundreds of millions
> of connections, massive broadcast channels, and E2EE, here's Stop 4 — each piece maps to a ceiling, and below that
> ceiling I'd remove it."*

---

## The 60-second Principal summary

> *"It's a two-plane system — stateful WebSocket gateways for live connections and a durable wide-column store that's
> the source of truth. I start with a single server holding sockets in a map, then split into gateways with a
> `user→gateway` registry so I can route across boxes, persisting every message before I deliver it and assigning a
> per-conversation sequence number for ordering, gap detection, and reconnect sync — with a client-generated message id
> for idempotent dedup, so at-least-once plus dedup gives exactly-once effect. That's the base. Pushed to global scale I
> climb one lever at a time: a pub-sub bus and dedicated fan-out workers for groups, then a consistent-hash sharded
> connection tier when a flat fleet strains, multi-region gateways with geo-routing when latency and data residency
> demand it, broadcast channels flipped from push to pull when groups hit hundreds of thousands, a dedicated
> per-conversation sequencer if the store can't give me monotonicity, and end-to-end encryption when the product
> requires it — knowing E2EE removes server search and moderation. Each piece has a named trigger, and below it I'd
> remove it. Presence and receipts stay best-effort throughout — I shed them before I ever drop a message."*

← Back to **[Chat System overview](README.md)** · Base **[deep dives](deep-dives.md)** · **[cheatsheet](cheatsheet.md)**
