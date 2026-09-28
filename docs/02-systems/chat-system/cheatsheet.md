# Chat / Messaging System — Cheatsheet

> One-page revision. Reproduce from memory and you can drive the interview.

---

## Problem
Real-time 1:1 + group messaging (WhatsApp/Messenger/Slack): sub-second delivery, presence, receipts, multi-device
history, offline push — at hundreds of millions of concurrent connections, tens of billions of messages/day.

## Profile (say first)
**Connection-bound, not request-bound.** Stateful real-time plane (WebSocket gateways) + durable storage plane (source
of truth). You pay for who's *connected*, not who's *talking*.

## Requirements
- **Must:** real-time 1:1 + group, durable multi-device history, ordered at-least-once delivery + dedup, presence,
  receipts, offline push.
- **Defer/seams:** large-group/broadcast fan-out, media (object store + URL), E2EE (a mode), server search.
- **Dominant NFRs:** massive concurrent connections · low-latency ordered delivery · no message loss · multi-device sync.
  Presence/receipts explicitly softer.

## Numbers (example)
| | |
|---|---|
| Concurrent connections | ~200M → **~1k stateful gateways**, sized by RAM/conn (~2 TB) — *the* cost |
| Heartbeats | ~6.6M frames/sec before any real message |
| Messages | 50B/day → ~500k/s avg, ~2.5M/s peak (receipts add ~2×) |
| Fan-out | 1 write → up to millions of deliveries (groups); **delivery ≫ send** |
| Storage | ~15 TB/day, ~5.5 PB/yr → wide-column by conv_id, tiered retention |

## Architecture
```text
clients ── WebSocket ──► [Gateways] (stateful, sticky, hold sockets, heartbeat, register user→gateway)
                              │ send
                         [Message Service] dedup(clientMsgId) · assign SEQ · PERSIST · resolve members
                              │
             ┌────────────────┼─────────────────┐
       [Routing/PubSub]  [Message Store]   [Offline?] → [Notification System] (push)
        user→gateway      wide-column
        (Redis, TTL)      partition=conv_id, cluster=seq
             │
       recipient [Gateways] ── push over socket ──► clients ── receipt(delivered/read) ──►
```

## Key ideas
- **Two ids run everything:** server **per-conversation `seq`** (ordering + gap detection + sync cursor) and client
  **`clientMsgId`** (idempotent dedup). At-least-once + dedup = **exactly-once effect**.
- **Persist before deliver** — message is safe even if the gateway dies; connection tier is **disposable**.
- **Routing:** `user→gateway` registry (Redis, ephemeral, TTL) + **pub-sub bus**; stale entry → fall back to push.
- **Receipts:** cumulative (`upTo: seq`) → idempotent, self-healing, softer/batchable lane. Cap per-member read
  receipts in big groups.
- **Presence/typing:** best-effort, TTL-derived from connections, never persisted, **shed first** under load.
- **Multi-device:** `user → many connections`; per-device read cursors converge on **max**; sync-after-cursor on reconnect.
- **Groups:** small = **fan-out on write**; huge/broadcast = **fan-out on read / pull**.

## Transport
WebSocket (full-duplex, low overhead) + long-poll fallback. Auth on connect, not per message. Sticky sessions.

## Bottlenecks (ranked)
1. **Connection fleet** → scale gateways horizontally + lean bytes/conn
2. **Routing/fan-out** → pub-sub bus + dedicated fan-out workers
3. **Hot conversation/partition** → queue + pace; flip broadcast to pull
4. Message-store writes → partition by conv_id; receipts softer lane
5. Reconnect thundering herd → backoff+jitter + accept rate-limiting

## Failure one-liners
- Client/gateway drop → persisted already → reconnect + **cursor sync** (never lost).
- Store write fails → **don't ack** → client retries same clientMsgId (idempotent).
- Stale registry → delivery miss → **push + sync** (not loss).
- Dup → dedup on clientMsgId. Gap → detect by seq, replay from store, render by seq.
- Hot group → fan-out workers + queue; broadcast → pull. Offline → push + sync.
- Presence/receipts → best-effort, self-healing, **shed before messages**.

## Patterns used
[websockets/real-time](../../01-patterns/websockets-realtime.md) · [queues+workers](../../01-patterns/queues-workers.md) ·
[idempotency](../../01-patterns/idempotency.md) · [sharding](../../01-patterns/sharding.md) ·
[caching](../../01-patterns/caching.md) · [rate limiting](../../01-patterns/rate-limiting.md) ·
[notifications](../notification-system/README.md) (offline push).

## SQL vs NoSQL
Messages → wide-column by conv_id + seq. Conversations/members → relational/wide-column. Read cursors → KV.
Routing registry → in-memory Redis (ephemeral, TTL).

## Summary line
> "Two planes: stateful WebSocket gateways for live connections + a durable wide-column store that's truth. On send I
> dedup on a client id, assign a per-conversation sequence number, persist, then route to the recipient's gateway via a
> Redis registry + pub-sub — persist before deliver. At-least-once + dedup = exactly-once effect; the seq number is
> ordering, gap detection, and the reconnect sync cursor. Small groups fan out on write, broadcasts flip to pull.
> Presence/receipts are best-effort; offline gets a push + cursor sync. The connection tier is disposable behind the store."

← Back to **[README](README.md)**
