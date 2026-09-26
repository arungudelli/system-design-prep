# Real-Time Delivery (Polling → Long-Poll → SSE → WebSockets)

> "Real-time" is a **staircase of transports**, not a single choice. The interview signal isn't "I'd use
> WebSockets" — it's knowing the four rungs (**polling, long polling, SSE, WebSockets**), naming the **trigger**
> that forces you up a rung, and being able to answer *"when is a simple poll enough, and what requirement
> justifies a persistent connection?"* — then handling the genuinely hard part: **fanning a message out to the
> right connection when that connection lives on one of 10,000 servers.**

**One-line mental model:** the client wants a server-pushed update, but HTTP is client-pull. Each rung trades
**more infrastructure and stateful connections** for **lower latency and true server-push** — climb only when a
latency or efficiency requirement forces it.

---

## Problem

HTTP is **request/response**: the client asks, the server answers, the connection closes. But many features need
the **server to tell the client** something happened — a new chat message, a price tick, a ride's location, a
"your export is ready" toast, a collaborator's cursor. There's no channel for the server to initiate. The naive
fix (ask again and again) wastes requests and still lags; the powerful fix (hold a connection open) turns your
**stateless** web tier into a **stateful** one, which is where all the real difficulty lives.

## When to use (a persistent/real-time transport)
- The server must **push** data the client didn't ask for *right now* (chat, notifications, live feeds).
- **Low latency** matters — sub-second updates, not "next time you refresh."
- Updates are **frequent** enough that repeated polling would waste huge numbers of requests.
- **Bidirectional**, chatty interaction (chat, collaborative editing, multiplayer, live location).

## When NOT to use
- Data changes **rarely** or the user only needs it **on demand** → a plain request (or a slow poll) is simpler
  and cheaper. Don't hold a million idle sockets for data that changes hourly.
- **One-directional, infrequent** server updates → **SSE** or long-poll, not full WebSockets.
- You can tolerate **seconds of staleness** → short polling is trivial, stateless, and CDN/proxy-friendly.
- You'd add a stateful connection tier for a feature that a **10-second poll** would satisfy → complexity with no
  matching requirement. **Start at the lowest rung that meets the latency budget.**

---

## The staircase (four rungs — climb one lever at a time)

```text
  latency ↓ / infra ↑
  ▲
  │  4. WebSocket   full-duplex, persistent TCP; server↔client both push; lowest latency
  │  3. SSE         one-way server→client stream over one HTTP conn; auto-reconnect; text only
  │  2. Long poll   client asks, server HOLDS the request until data (or timeout), then client re-asks
  │  1. Polling     client asks every N seconds regardless; stateless, simplest, laggy + wasteful
  ▼
```

| Rung | Direction | Latency | Connection | Cost / complexity | Reach for it when… |
|---|---|---|---|---|---|
| **1. Short polling** | client-pull | Up to N s stale | None (normal requests) | **Lowest** — stateless, cacheable, trivial | Updates are infrequent / staleness of a few seconds is fine |
| **2. Long polling** | server→client (emulated) | ~immediate | Held request, re-opened each cycle | Low-ish; works everywhere | You need push, but can't run WebSockets (old proxies, simple infra) |
| **3. SSE** | **one-way** server→client | ~immediate | 1 long-lived HTTP conn | Medium; built-in **auto-reconnect + `Last-Event-ID`** | Server pushes a **stream** (feeds, notifications, tickers); client rarely sends |
| **4. WebSocket** | **full-duplex** | Lowest | 1 persistent TCP (upgraded from HTTP) | **Highest** — stateful tier, LB support, scaling | **Bidirectional, chatty, low-latency** (chat, collab, gaming, live location) |

### The one-liner for each trigger
- **Polling → long polling:** "Polling either wastes requests (poll fast) or is stale (poll slow). Long polling
  gives near-real-time push without a persistent protocol — the server just holds the request until there's data."
- **Long polling → SSE:** "If the flow is **server→client only** and streaming, SSE is cleaner: one connection,
  the browser auto-reconnects and resumes with `Last-Event-ID`, and I don't reopen a request per message."
- **SSE → WebSockets:** "The moment the client also needs to send frequently — chat, typing indicators,
  collaborative edits — I want **full-duplex**. That's WebSockets."

> **Interview line:** *"I default to the lowest rung that meets the latency budget. A poll is fine for something
> that changes every few minutes. I climb to SSE for one-way streams and to WebSockets only when the interaction
> is bidirectional and chatty — because a persistent connection turns my stateless web tier stateful, and that's
> a real cost I only pay when a requirement demands it."*

---

## Polling vs long polling (the cheap rungs, in detail)

- **Short polling** — `GET /messages?since=…` every N seconds. Stateless, load-balances trivially, cache/proxy
  friendly, dead simple. **Downsides:** average latency ≈ N/2; most polls return "nothing new" → wasted requests
  and DB reads at scale. Tune N to the staleness you can tolerate; add jitter so clients don't synchronize into a
  thundering herd.
- **Long polling** — client sends a request; the **server holds it open** until data arrives or a timeout (say
  30 s), then responds; the client immediately re-requests. Near-real-time with plain HTTP. **Downsides:** a held
  request ties up a server thread/connection (fine on async servers, costly on thread-per-request), reconnect
  overhead each cycle, and you must dedupe/order across cycles with a cursor (`since`/`Last-Event-ID`).

---

## WebSockets & SSE (the persistent rungs, in detail)

- **WebSocket** — starts as an HTTP request with `Upgrade: websocket`; after the handshake the same TCP
  connection becomes a **full-duplex** channel carrying lightweight frames both ways with almost no per-message
  overhead. Binary or text. This is the workhorse for chat, collaboration, gaming, live location.
- **SSE (Server-Sent Events)** — a single long-lived HTTP response the server keeps writing to (`text/event-stream`).
  **One-way** (server→client), **text only**, but it's simple, rides ordinary HTTP/2, and the browser's
  `EventSource` gives you **automatic reconnect** and **`Last-Event-ID`** resume for free. Pair it with normal
  `POST`s for the occasional client→server message and you've covered many "live feed" cases without WebSockets.

> **Nuance worth saying:** *"SSE isn't 'worse WebSockets' — for a pure server-push stream it's often the better
> choice because reconnection and resume are built in and it's just HTTP. I reserve WebSockets for when the client
> also talks back frequently."*

---

## The connection tier (where the real design is)

A persistent-connection feature splits your backend into two very different tiers:

```text
                       ┌──────────────── stateful ────────────────┐
  Clients ──WSS──►  Load Balancer  ──►  [ Gateway / WS servers ]  ──► Pub/Sub (Redis / Kafka)
   (sticky by         (L4 or L7,          each holds a subset of        │  broadcast bus
    conn id)           sticky)             live connections;            ▼
                                           local registry conn→user   [ Backend services / DB ]
                                                     ▲                    write, then publish
                                                     └── message for user U? find U's server, deliver
```

- **Gateway / WebSocket servers** — their *only* job is holding connections and moving frames. Keep business
  logic **out** so you can scale and restart them independently. Each server holds a **subset** of all live
  connections and knows, locally, `connectionId → userId`.
- **The hard problem — fan-out / routing:** a message for user *U* arrives at some backend, but *U*'s socket is
  pinned to **one specific gateway** among thousands. How do you get the frame there? Two patterns, usually combined:
  - **Pub/Sub broadcast (simple, default):** every gateway subscribes to a bus (Redis Pub/Sub, Kafka). Publish
    the message; each gateway checks "do I hold a connection for *U*?" and delivers if so. Simple, no lookup, but
    **every gateway sees every message** — fine to moderate scale, wasteful at the very top.
  - **Connection registry (targeted):** a shared store (Redis) maps `userId → gatewayId`. Look it up, deliver
    only to that gateway. Scales fan-out but adds a lookup and a registry to keep consistent on connect/disconnect.
  - **Real answer at scale:** route by **channel/room**, not per-user — gateways subscribe only to the rooms they
    hold members for; a message to a room hits only the relevant gateways.

> **Interview line:** *"The connections are stateful and spread across many servers, so the core problem is:
> given a message for a user, find the server holding that user's socket. I'd start with Redis Pub/Sub broadcast
> for simplicity, and move to a `user→server` registry (or room-based subscriptions) when broadcasting every
> message to every gateway becomes the bottleneck."*

---

## Sticky sessions & load balancing (a mandatory gotcha)

- A WebSocket is a **single long-lived TCP connection to one server**. Ordinary round-robin load balancing
  across requests **doesn't apply** — once established, all frames must reach **that** server.
- The **handshake** (and any long-poll/SSE reconnect) must land on a server that can serve the connection, so LBs
  use **sticky sessions / connection affinity**. L4 (TCP) LBs keep the socket pinned naturally; L7 LBs must be
  configured to support the `Upgrade` handshake and affinity.
- **Consequence — deploys and failures are disruptive:** restarting a gateway drops every socket it holds;
  thousands of clients reconnect at once (a **reconnect storm**). Mitigate with **graceful drain** (stop taking
  new connections, let clients migrate), **staggered rollouts**, and **client-side reconnect with backoff + jitter.**
- Because connections are stateful, **capacity planning is about concurrent connections**, not requests/sec — see
  the connection-count math below.

---

## Connection lifecycle: heartbeats, reconnection, resume

A persistent connection can die **silently** — a laptop sleeps, wifi drops, a NAT times out — and neither side
gets a clean `close`. You must detect and recover.

- **Heartbeats (ping/pong):** each side periodically pings; no pong within a window → assume dead, close, and let
  the client reconnect. This also keeps NATs/proxies from reaping an "idle" connection. Without heartbeats you
  accumulate **zombie connections** (server thinks the user is online; they left 20 minutes ago).
- **Reconnection:** the client must reconnect automatically with **exponential backoff + jitter** (never a tight
  loop — that's a self-inflicted DDoS on your gateways after an outage). SSE's `EventSource` does this for you.
- **Resume / no gaps:** on reconnect, the client sends its **last-seen cursor** (`Last-Event-ID` / last message
  id / sequence number) and the server replays what was missed. This is what makes reconnection **seamless**
  rather than lossy — pair it with per-conversation sequence numbers so the client can also **detect gaps** and
  reorder.

> **Interview line:** *"I assume every connection will die silently, so I add heartbeats to detect dead sockets,
> reconnect with backoff and jitter to avoid a reconnect storm, and resume from a last-seen sequence id so no
> messages are lost across the gap."*

---

## Delivery guarantees over a flaky connection

A frame written to a socket is **not** a guarantee it was received (the connection may have just died). For
anything that matters:

- **Application-level ACKs:** the receiver acks by message id; unacked messages are **retried on reconnect**.
  (TCP delivery ≠ application delivery.)
- **Persist first, then push:** for chat/notifications, **write the message to the store** before/independently of
  pushing it. The socket is a fast-path *optimization*; the store is the source of truth the client syncs against.
- **Ordering:** use a **per-conversation/room sequence number** so clients order correctly and detect gaps
  regardless of network reordering or multi-path delivery.
- **Idempotency:** retries mean the same message may arrive twice → dedupe by message id on the client (and
  server). See [idempotency](idempotency.md).
- **Offline → fall back to push:** if the user has **no live connection**, route through the
  [notification system](../02-systems/notification-system/README.md) (APNs/FCM/email) instead of dropping it.

---

## Presence ("who's online") — deceptively hard

- **Naive:** mark a user online on connect, offline on disconnect. Breaks the instant a connection dies silently
  (they show "online" forever) or flaps (rapid on/off).
- **Better:** presence is a **heartbeat + TTL** in a fast store (Redis). Refresh a `user:online` key with a short
  TTL on each heartbeat; if it isn't refreshed, it **expires** → offline. Self-healing without a clean disconnect.
- **Fan-out cost:** broadcasting every presence change to every friend is **O(connections × friends)** — brutal
  at social-network scale. Batch/debounce updates, only push presence for people actually being viewed, and accept
  that presence is **approximate and eventually consistent** ("last seen 2 min ago" is fine).

> **Interview line:** *"I model presence as a heartbeat-refreshed key with a TTL rather than online/offline flags,
> so a silently dropped connection self-heals to offline. Presence is best-effort and eventually consistent — I
> won't pay for strong consistency on 'is my friend online.'"*

---

## Multi-device & multi-connection

One user = **many simultaneous connections** (phone, laptop, tablet). Consequences:

- Map **`userId → set of connections`**, not one. A message to a user **fans out to all their devices**.
- **Read/state sync:** marking a message read on the phone must propagate to the laptop → send sync events across
  all of the user's connections.
- **Presence** = online if **any** device has a live connection.
- The `user→server` registry becomes `user→{server per connection}`; route to each.

---

## Scaling the connection tier

- **It's about concurrent connections, not RPS.** A single well-tuned server can hold **tens to hundreds of
  thousands** of idle connections (the classic **C10K → C10M** problem), bounded by file descriptors, memory per
  connection, and event-loop efficiency. Plan the fleet from **peak concurrent connections**, not requests/sec.
- **Quick math:** 10M concurrent users ÷ ~100k connections/server ≈ **~100 gateway servers** just to hold
  sockets (before message throughput). Say this out loud — it shows you know the tier is sized differently.
- **Keep gateways thin & stateless-ish:** hold connections + a local registry only; push all business logic and
  durable state behind them so gateways can be scaled and recycled freely.
- **Backpressure:** a slow client can't keep up with a fast stream → bound per-connection send buffers and
  **drop/coalesce** or disconnect abusers rather than blowing up memory. (Same discipline as
  [queues & workers](queues-workers.md) backpressure, applied per-socket.)

---

## Trade-offs (argue both sides)

- **Stateless (polling) vs stateful (WebSocket/SSE):** stateless scales and load-balances trivially and survives
  deploys painlessly, but is laggy/wasteful; stateful gives true low-latency push at the cost of a whole
  connection-management tier (sticky LBs, heartbeats, reconnection, fan-out routing). **Only go stateful when
  latency/efficiency demands it.**
- **WebSocket vs SSE:** WebSocket = full-duplex + binary, but you build reconnect/resume yourself and need LB
  support; SSE = one-way + text, but reconnect/resume are free and it's just HTTP. Pick by **directionality**.
- **Pub/Sub broadcast vs connection registry:** broadcast is simple but every gateway sees every message;
  registry/room-routing scales fan-out at the cost of a lookup and consistency on connect/disconnect.
- **Push socket vs durable store:** the socket is the fast path; the store is truth. Don't treat "wrote to socket"
  as "delivered" — persist and ack.
- **Presence accuracy vs cost:** strong, instant presence is expensive at fan-out; accept approximate/eventual.

Full comparison in **WebSocket vs SSE vs polling** *(trade-off doc coming)*.

---

## Where it appears (across systems)

- **Chat / messaging** — the canonical case: WebSockets, per-conversation sequence numbers, presence, multi-device,
  offline → push. *(system coming next)*
- **Notification system** — in-app real-time delivery via a live connection, with **fallback to push/email** when
  the user is offline. See [notification system](../02-systems/notification-system/README.md).
- **Ride sharing / live location** — high-frequency driver→rider location streams; often SSE or WebSockets with
  heavy geospatial fan-out.
- **Collaborative editing / multiplayer** — full-duplex, low-latency, ordering-sensitive.
- **Live dashboards / tickers / feeds** — one-way server→client → **SSE** is often the right rung.

---

## Quick checklist

- [ ] What's the **latency budget**? Pick the **lowest rung** that meets it (poll → long-poll → SSE → WebSocket).
- [ ] **Direction:** one-way server→client → **SSE**; bidirectional & chatty → **WebSocket**.
- [ ] **Load balancing:** sticky sessions / connection affinity; L7 configured for the `Upgrade` handshake.
- [ ] **Heartbeats** (ping/pong) to kill zombie connections and keep NATs open.
- [ ] **Reconnect** with exponential backoff + **jitter**; **resume** from a last-seen cursor (`Last-Event-ID`).
- [ ] **Fan-out routing:** Redis Pub/Sub broadcast (simple) → `user/room → server` registry (scales).
- [ ] **Delivery:** app-level **ACKs**, **persist-then-push**, per-conversation **sequence numbers**, dedupe.
- [ ] **Offline** users → fall back to the **[notification system](../02-systems/notification-system/README.md)**.
- [ ] **Presence** = heartbeat + **TTL** (self-healing), best-effort/eventual; watch fan-out cost.
- [ ] **Multi-device:** `user → set of connections`; fan out to all; sync read state.
- [ ] **Capacity** from **peak concurrent connections**, not RPS; graceful drain on deploy to avoid reconnect storms.

---

← Back to **[patterns](../index.md)** · Related: [queues & workers](queues-workers.md) · [idempotency](idempotency.md) · [rate limiting](rate-limiting.md) · [notification system](../02-systems/notification-system/README.md)
