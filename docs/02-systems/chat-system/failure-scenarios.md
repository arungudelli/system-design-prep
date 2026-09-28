# Chat / Messaging System — Failure Scenarios

> Themes: **connections die constantly** (flaky mobile networks are the normal case, not the exception), **persist
> before deliver so a message is never lost**, **never duplicate or reorder**, and **degrade presence/receipts before
> ever dropping a message**. The store is truth; the socket is disposable.

---

## A client's connection drops (the normal case)

- **Impact:** in-flight messages may not have reached the client; the socket is gone.
- **Detect:** missed heartbeats → the gateway **reaps** the socket; presence flips to offline via TTL.
- **Contain:** the message is **already persisted** (persist-before-deliver). Nothing is lost.
- **Recover:** client reconnects (with **backoff + jitter**), re-auths, sends `sync{cursors}`; server **replays
  everything after the client's last seq**. The sequence number makes catch-up exact. See
  [multi-device sync](deep-dives.md#6-multi-device-sync).

## A connection gateway crashes

- **Impact:** every socket on that box (up to ~1M connections) drops at once.
- **Detect:** health checks; registry entries for that gateway go stale/TTL out.
- **Contain:** messages persisted before the crash are safe; undelivered ones become **offline** → push + reconnect
  sync. The gateway held **no source-of-truth state** — only sockets and an ephemeral registry — so its loss is
  recoverable by design.
- **Recover:** clients reconnect and are **spread across surviving gateways** (sticky/consistent-hash routing);
  registry rebuilds from the reconnects.

## Reconnect thundering herd

- **Impact:** a gateway/region drop makes millions of clients reconnect **simultaneously** → a connect-storm that can
  topple the survivors.
- **Contain:** **client-side backoff + jitter** (never reconnect in lockstep), **server-side accept rate-limiting**,
  and spreading reconnects across the fleet. Capacity headroom sized for herd, not steady state. See
  [bottlenecks](architecture.md#11-bottlenecks-ranked).

## Routing registry is stale (points to a dead gateway)

- **Impact:** delivery is routed to a gateway that no longer holds the socket → push fails.
- **Contain:** treat a failed delivery as "recipient offline" → **fall back to push notification**; the recipient syncs
  on reconnect. The registry is ephemeral + TTL'd precisely so staleness is a *routing miss*, not data loss. See
  [routing](deep-dives.md#1-connection-routing).

## Message store write fails / is slow

- **Impact:** can't persist → can't safely ack.
- **Contain:** **do not ack** on write failure. The sender's client retries with the **same `clientMsgId`** → when the
  write eventually succeeds it's **idempotent** (no duplicate). Better a spinner than a lost or duplicated message.
- **Recover:** partition by `conversation_id` isolates the blast radius; a hot/slow partition affects one conversation,
  not the fleet.

## Duplicate messages

- **Impact:** the same message appears twice (retries, at-least-once redelivery, resend after flaky reconnect).
- **Contain:** **dedup on `clientMsgId`** — a re-sent message returns the original `ack`/`seq` instead of a second
  write. At-least-once transport + client-id dedup = exactly-once **effect**. See
  [ordering & dedup](deep-dives.md#2-ordering--deduplication).

## Lost or out-of-order messages (gaps)

- **Impact:** a client is missing seq 1024, or receives 1025 before 1024.
- **Contain:** the client **detects the gap by sequence number** (has 1023, sees 1025 → requests 1024) and **renders
  by seq, not arrival time**. The store can always replay the missing range. Ordering is a client-side sort keyed on a
  server-assigned seq — the network can reorder all it wants.

## Hot conversation / hot partition (giant or celebrity group)

- **Impact:** a huge, chatty group concentrates writes + fan-out on one `conversation_id` partition and floods the
  router.
- **Contain:** **dedicated fan-out workers + a queue** absorb the burst (don't fan out inline); **cap group size** or
  **flip broadcast channels to pull** (store once, clients pull on open); consider sub-partitioning a mega-group's
  stream. See [group fan-out](deep-dives.md#5-group-fan-out).

## Recipient offline

- **Impact:** no live socket to push to.
- **Contain:** message is persisted → enqueue a **[push notification](../notification-system/README.md)**; deliver the
  missed messages via **cursor sync** on reconnect. Sender sees "sent," upgraded to "delivered" when the recipient
  returns. Delayed, never dropped.

## Receipts / presence storm or loss

- **Impact:** receipt/presence volume spikes, or some are lost.
- **Contain:** both are **softer lanes** — **cumulative receipts** (`upTo: seq`) self-heal (a lost receipt is fixed by
  the next), presence is **best-effort + TTL** (a missed update self-corrects on the next heartbeat). Under load, **shed
  presence/typing first** — never shed messages. Degrade the soft signals to protect the hard guarantee.

## Split brain / two gateways think they own a user

- **Impact:** a user briefly appears connected on two gateways (e.g. reconnect before the old socket is reaped) → a
  message could be double-routed.
- **Contain:** **dedup on `clientMsgId`/`seq` at the client** collapses a double-delivery; the stale registry entry
  TTLs out; last-writer-wins on the registry. Duplicate *delivery* is harmless because dedup makes it a no-op.

## Region failure

- **Impact:** lose gateways + local infra in a region.
- **Contain:** clients reconnect to another region (geo-routing + failover); the durable, replicated store means
  accepted messages survive; region-local registries rebuild on reconnect. Multi-region is a top-stop concern — see
  [principal deep dive](principal-deep-dive.md).

---

## Failure-handling toolkit used here

`persist-before-deliver` (never lose a message) · `sequence number` (ordering + gap detection + sync cursor) ·
`clientMsgId dedup` (no duplicates, idempotent retries) · `ephemeral TTL'd routing registry` (staleness = routing miss,
not loss) · `reconnect backoff + jitter + accept rate-limiting` (thundering herd) · `offline → push + cursor sync` ·
`fan-out workers + queue` (hot groups) · `cumulative receipts` (self-healing) · `best-effort presence` (shed first) ·
`partition by conversation_id` (blast-radius isolation).

## Priorities (say this)

> *"My one hard guarantee: never lose a message and never duplicate or reorder it. Everything serves that — I persist
> before I deliver, so a dropped connection or crashed gateway only delays delivery (via push + cursor sync), never
> drops it; I assign a per-conversation sequence number for order and gap detection; and I dedup on a client-generated
> message id so retries are idempotent. Presence and receipts are best-effort and I shed them first under load. The
> connection tier is disposable because the durable store is the source of truth."*

→ Next: **[Interview Questions](interview-questions.md)**
