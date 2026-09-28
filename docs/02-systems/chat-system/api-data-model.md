# Chat / Messaging System — API & Data Model

> Chat has **two API surfaces**: a **persistent WebSocket protocol** (the real-time plane — send, deliver, ack,
> receipts, presence, typing) and a **REST/HTTP surface** (the control plane — auth, load history, create groups,
> upload media). The data model is anchored by one idea: a **monotonic sequence number per conversation** that gives
> ordering, gap detection, and a sync cursor — plus **client-generated message IDs** for idempotent retries.

---

## The WebSocket protocol (the hot path)

A client opens **one** authenticated WebSocket to its assigned gateway and multiplexes everything over it.

### Connect & authenticate
```text
GET /ws  (Upgrade: websocket)
Authorization: Bearer <token>           # auth on connect; short-lived token
→ 101 Switching Protocols
← { "type": "connected", "sessionId": "s_9", "resumeCursors": { ... } }   # server hello
```
Auth happens **once at connect**, not per message (per-message auth would wreck latency). The socket carries the
authenticated identity for its lifetime.

### Send a message (client → server)
```text
→ { "type": "send",
    "conversationId": "c_42",
    "clientMsgId": "d3b0…",        # CLIENT-generated UUID → idempotency / dedup on retry
    "body": "hey",
    "media": null }                # or { "url": "...", "mime": "image/jpeg" } — media is uploaded via REST first

← { "type": "ack",                 # server acknowledges + assigns ordering
    "clientMsgId": "d3b0…",        # correlates to the send
    "serverMsgId": "m_88",
    "conversationId": "c_42",
    "seq": 1024,                   # per-conversation monotonic sequence number
    "ts": 1690000000 }             # server timestamp
```
The **`ack` is "persisted & ordered," not "delivered."** The client shows a single tick (sent) on `ack`. The
**`clientMsgId`** is the dedup key: if the client retries the same send after a flaky reconnect, the server returns
the *same* `ack` instead of writing a duplicate (see [ordering & dedup](deep-dives.md#2-ordering--deduplication)).

### Receive a message (server → client)
```text
← { "type": "message",
    "conversationId": "c_42",
    "serverMsgId": "m_88", "seq": 1024,
    "senderId": "u_7", "body": "hey", "ts": ... }

→ { "type": "receipt", "conversationId": "c_42", "upTo": 1024, "state": "delivered" }
  … later …
→ { "type": "receipt", "conversationId": "c_42", "upTo": 1024, "state": "read" }
```
Receipts are **cumulative** (`upTo: seq`) — "I have everything through seq 1024" — not one-per-message. That collapses
N receipts into one and makes them idempotent (see [receipts](deep-dives.md#3-delivery--read-receipts)).

### Presence & typing (ephemeral, best-effort)
```text
→ { "type": "typing", "conversationId": "c_42", "state": "start" | "stop" }
← { "type": "presence", "userId": "u_7", "state": "online" | "offline", "lastSeen": ... }
```
These are **never persisted** and may be dropped under load — see [presence](deep-dives.md#4-presence--typing).

### Heartbeat (keep-alive)
```text
→ ping    ← pong        # every ~30s; missing pongs → server reaps the dead socket
```
Local to the gateway; never touches the store (see [capacity](capacity.md#2-heartbeat--idle-overhead-the-hidden-load)).

### Sync after reconnect (the catch-up path)
```text
→ { "type": "sync", "cursors": { "c_42": 1000, "c_7": 55 } }   # last seq I have per conversation
← { "type": "message", ... seq: 1001 } … 1024                   # server replays the gap from the store
```
The client sends **its last-seen seq per conversation**; the server streams everything after it. This is how a phone
that was offline for an hour catches up — **the sequence number *is* the cursor.**

---

## REST / HTTP surface (the control plane)

```text
POST /v1/conversations                 # create a 1:1 or group; returns conversationId
POST /v1/conversations/{id}/members    # add/remove members (group admin)
GET  /v1/conversations/{id}/messages?before=<seq>&limit=50   # paginate history (backfill/scroll-up)
GET  /v1/conversations?updatedAfter=…  # list a user's conversations (inbox), newest activity first
POST /v1/media                         # presigned upload → object store; returns a URL to put in a message
GET  /v1/users/{id}/devices            # registered devices (multi-device)
```
Media never flows over the socket — the client **uploads to the object store via a presigned URL**, then sends a
message whose body carries the resulting URL. History scroll-back is a normal paginated REST read, not a socket op.

---

## Data model

### Access patterns first (design the schema around these)
1. **Append a message** to a conversation, assigning the next `seq` — very high write volume.
2. **Read the latest N messages** in a conversation (open a chat) — the dominant read.
3. **Sync**: read all messages in a conversation **after seq X** (reconnect catch-up).
4. **List a user's conversations** newest-first (the inbox screen).
5. **Per-device read state**: where has each of a user's devices read up to?
6. **Route**: given a recipient user, which gateway(s) hold their live connection(s)?

### `messages` — the source of truth (wide-column)
```text
PARTITION KEY: conversation_id
CLUSTERING KEY: seq   (monotonic per conversation, ascending)
  server_msg_id, client_msg_id, sender_id, body | media_url, ts, deleted?
```
- **Partition by `conversation_id`, cluster by `seq`** → "latest 50 in a conversation" and "everything after seq X"
  are both **single-partition range scans** — exactly access patterns #2 and #3. This is the whole reason for a
  wide-column store (Cassandra/HBase/ScyllaDB-style) over a naive `messages` table.
- `client_msg_id` is indexed/checked for **dedup** on write.
- A **giant group is a hot partition** — flagged and handled in [group fan-out](deep-dives.md#5-group-fan-out).

### `conversations` & `conversation_members`
```text
conversations:        conversation_id (PK) | type (1:1|group) | name | created_at | last_seq
conversation_members: conversation_id + user_id (PK) | role | joined_at | muted?
                      (secondary index: user_id → conversations)   # for the inbox list
```
- `conversation_members` answers both "who's in this group?" (fan-out target list) and, by `user_id`, "which
  conversations is this user in?" (the inbox).

### `read_cursors` — per user **per device**
```text
conversation_id + user_id + device_id (PK) | delivered_up_to (seq) | read_up_to (seq)
```
- Per-**device** cursors are what make **multi-device** converge: each device reports its own progress; "read" for the
  account is the **max** across devices (see [multi-device sync](deep-dives.md#6-multi-device-sync)).

### `user_connections` — the routing registry (ephemeral, hot)
```text
user_id → { device_id → gateway_id, connected_at }      # in Redis / in-memory, TTL'd by heartbeat
```
- The map from **user → the gateway(s) holding their socket(s)**. Read on **every** message delivery to route it.
  Ephemeral (rebuilt on reconnect), TTL'd, **never the durable source of truth**. Detailed in
  [routing](deep-dives.md#1-connection-routing).

### `inbox` (optional — only if you fan-out on write for groups)
```text
user_id (PK) + conversation_id + seq   # a per-user materialized view of "conversations with new activity"
```
- Whether this exists at all is the **fan-out-on-write vs on-read** trade-off — see [trade-offs](tradeoffs.md#3-group-delivery-fan-out-on-write-vs-fan-out-on-read).

---

## SQL or NoSQL? (per store)

- **`messages` → NoSQL wide-column**, partitioned by `conversation_id`, clustered by `seq`. Write-heavy, enormous,
  and read as "recent-N / after-X in one conversation" — the wide-column sweet spot. A single relational table can't
  hold petabytes of messages with this read pattern cheaply.
- **`conversations` / `members` → relational or wide-column**; modest size, relational is fine (membership is
  a graph-ish, transactional-ish concern).
- **`read_cursors` → KV/wide-column**, keyed by conversation+user+device; tiny per row, huge in count, simple upsert.
- **`user_connections` → in-memory KV (Redis)**, ephemeral + TTL. Never a durable DB — it changes on every
  connect/disconnect.

> **Interview line:** *"Messages go in a wide-column store partitioned by conversation_id and clustered by a
> per-conversation sequence number, so 'latest 50' and 'everything after seq X' are single-partition scans — that
> sequence number is simultaneously my ordering guarantee, my gap-detector, and my sync cursor. The user→gateway
> routing map is ephemeral in Redis, never the source of truth. And I dedup on a client-generated message id so
> retries are idempotent."*

→ Next: **[Architecture](architecture.md)**
