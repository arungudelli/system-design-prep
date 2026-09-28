# Chat / Messaging System — Capacity Estimation

> See the method in **[Capacity estimation](../../00-framework/capacity-estimation.md)**. Chat breaks the habit of
> every stateless system so far: the dominant number is **concurrent live connections**, **not** requests/sec. A
> mostly-idle user still costs you a held socket + memory + a heartbeat. So we size the **connection fleet** first,
> then message throughput, then **fan-out amplification** (groups), then storage. The one-line story: **you pay for
> who is *connected*, not for who is *talking*.**

---

## Assumptions (state them)

- **2B registered users**, **500M DAU**.
- **Peak concurrent connections: ~200M** live sockets (people leave chat apps open all day → concurrency is high
  relative to DAU).
- **50B messages/day** sent (tens of billions).
- **Multi-device:** avg **~1.5 connections/user** online (phone + web/desktop).
- **Groups:** most chats are 1:1 or small; a long tail of **large groups (up to ~1k–10k members)** and
  **broadcast channels** (100k+).
- Peak ≈ **5×** average.

---

## 1. Connection fleet (the dominant cost — size this first)

```text
Peak concurrent connections ≈ 200M
Connections one tuned gateway can hold ≈ ~500k–1M  (C10M-class, epoll/kqueue, tuned kernel)
→ gateways ≈ 200M / ~500k ≈ 400  (then ×2–3 for headroom, deploys, AZ spread → ~1,000)
```

Memory is the real limit per box, not CPU:
```text
Per connection: socket buffers + TLS + app state ≈ ~10 KB (lean) … 50 KB (fat)
200M × 10 KB ≈ 2 TB RAM across the fleet   (just to hold idle sockets)
```

→ **So what?** The gateway tier is sized by **concurrent connections and RAM/connection**, and it exists *even at
zero messages*. This is why the connection tier is a **separate, horizontally-scaled, stateful fleet** with
[sticky sessions](../../01-patterns/websockets-realtime.md#sticky-sessions) — the first and biggest box you draw.
Driving down bytes/connection (lean protocol, shared buffers) is a top-line cost lever.

---

## 2. Heartbeat & idle overhead (the hidden load)

```text
200M connections, heartbeat every ~30 s to detect dead sockets
→ 200M / 30 ≈ ~6.6M heartbeat frames/sec  — before a single real message is sent
```

→ **So what?** At this scale **keeping connections alive is itself a workload.** Heartbeat interval is a direct
tuning knob (too short → wasted CPU/battery/bandwidth; too long → slow dead-connection detection, [zombie
sockets](../../01-patterns/websockets-realtime.md#heartbeats-detecting-dead-connections)). Heartbeats stay **local
to the gateway** — they must never touch the message store.

---

## 3. Message throughput

```text
50B messages/day ÷ 10^5 s ≈ ~500,000 messages/sec  (average)
peak ≈ 500k × 5 ≈ ~2.5M messages/sec
```

Each sent message is more than one write — count the amplifiers:
```text
1 message → 1 durable store write (source of truth)
          + routing lookup + push to N recipient devices
          + receipt events: delivered, read   (small metadata writes, ~2× the message count)
```

→ **So what?** ~2.5M msg/sec peak is large but tractable for a partitioned wide-column store; the sneaky multiplier
is **receipts** (delivered/read are extra write+deliver flows) and **fan-out** (next). Size the store for
**message writes + receipt writes**, and treat receipts as a softer, batchable lane.

---

## 4. Fan-out amplification (groups turn 1 write into N deliveries)

```text
1:1 message      → 1 store write, deliver to ~1.5 devices        (amplification ≈ 1×)
group of 100     → 1 store write, deliver to ~150 connections    (≈ 100×)
group of 10,000  → 1 store write, deliver to ~10k+ connections   (≈ 10,000×)
broadcast 1M     → 1 store write, deliver to up to 1M connections (celebrity/channel problem)
```

→ **So what?** **Delivery volume ≫ send volume**, and it's driven entirely by group size. Small groups fan out
inline; **large groups/broadcast are a different regime** — a single hot conversation can saturate one partition
and flood the router. This is the seam for a **dedicated fan-out stage + buffered
[queues](../../01-patterns/queues-workers.md)**, and it's why "how big can a group get?" is the highest-leverage
clarifying question (see [requirements](requirements.md#2-clarifying-questions)).

---

## 5. Presence fan-out (cheap per event, big in aggregate)

```text
User comes online → notify their contacts who are online.
500M DAU flipping online/offline/away, avg ~100 contacts each,
often batched, but the raw ceiling is large and bursty (morning login spikes).
```

→ **So what?** Presence is high-frequency and **best-effort** — this is why it's **ephemeral (TTL'd, never durably
stored)**, aggressively **batched/debounced**, and allowed to be slightly stale. Trying to make presence strongly
consistent would cost more than messaging itself.

---

## 6. Storage

```text
Messages: 50B/day × ~300 bytes (text + metadata) ≈ ~15 TB/day
          × 365 ≈ ~5.5 PB/year   (before replication)
Receipts/status: ~2× message count × ~50 bytes ≈ a few TB/day
Media: NOT in the message store — object store + CDN; message carries a URL
Presence: ephemeral, in-memory/TTL — effectively 0 durable
```

→ **So what?** Message history is **petabyte-scale and write-heavy with a "recent-N" read pattern** → a
**partitioned wide-column store keyed by `conversation_id`, clustered by sequence/time** (one-partition reads for
"latest 50"), with **tiered retention** (hot recent messages, cold/archived old ones — object storage). Keeping
media *out* of the message store is what keeps these numbers sane.

---

## 7. Bandwidth (sanity check)

```text
Ingress (sends):  50B × ~300 B ≈ ~15 TB/day  ≈ ~1.4 Gbps average
Egress (delivery): ingress × fan-out factor → dominated by groups/broadcast, can be many× ingress
```

→ **So what?** Text is small; **fan-out, not payload size, drives egress.** Media bypasses the socket entirely
(CDN), which is the only reason a text-shaped bandwidth budget holds.

---

## Summary — what the numbers told us

| Number | Value | Design consequence |
|---|---|---|
| **Concurrent connections** | ~200M peak | **Stateful gateway fleet** sized by connections + RAM/conn (~1k boxes) — *the* dominant cost |
| Bytes/connection | ~10–50 KB | Lean protocol / shared buffers is a top cost lever (~2 TB RAM) |
| Heartbeats | ~6.6M frames/sec | Keep-alive is its own load; heartbeat interval is a knob, kept **local** |
| Message rate | ~500k/s avg, ~2.5M/s peak | Partitioned wide-column store; receipts add ~2× small writes |
| Fan-out | 1 write → up to millions of deliveries | **Delivery ≫ send**; dedicated fan-out stage + queues for large groups |
| Presence | high-freq, bursty | Ephemeral + TTL + batched, **best-effort** |
| Message storage | ~15 TB/day, ~5.5 PB/yr | Wide-column by `conversation_id`, clustered by seq/time; **tiered retention** |
| Egress | fan-out-dominated | Media off-socket via CDN; text budget only holds because of that |

The profile: **connection-bound, not request-bound** — a huge stateful fleet holding mostly-idle sockets, moderate
steady message QPS with **fan-out amplification** on groups and **best-effort ephemeral presence** layered on top.
**Design for concurrency and fan-out, not for average send rate.**

→ Next: **API & Data Model** *(coming)*
