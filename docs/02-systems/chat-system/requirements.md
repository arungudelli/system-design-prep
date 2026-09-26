# Chat / Messaging System — Requirements

## 1. Problem statement (the interview prompt)

> "Design a real-time chat system like WhatsApp / Messenger / Slack. Users exchange messages **1:1 and in groups**,
> in real time, see who's **online** and when messages are **delivered/read**, get their **history on any device**,
> and receive a **push notification** when they're offline."

Deliberately broad. The three answers that reshape everything: **1:1 vs large groups**, **how strong the ordering/
delivery guarantee must be**, and **whether it must be end-to-end encrypted**. Clarify before drawing boxes.

---

## 2. Clarifying questions

### Product / scope
- **1:1 only, or group chat too? How large can a group get?** 
  *Why:* small groups are a trivial fan-out; **large groups (10k+) / broadcast channels** are a different system
  (fan-out amplification, hot partitions) — this single answer changes the architecture more than any other.
- **Which receipts?** sent → delivered → read? "Typing…" indicators?
  *Why:* each receipt is an extra message flow and write; read receipts across a big group are their own fan-out.
- **Media** (images, video, files, voice) or **text-only** first?
  *Why:* media doesn't flow over the socket — it's an object-store upload + a message carrying a URL. Different path.
- **Message history & search?** Kept forever? Server-side search?
  *Why:* retention drives storage cost; server-side search conflicts with **end-to-end encryption**.
- **Multi-device?** One account on phone + web + desktop simultaneously?
  *Why:* forces `user → many connections`, per-device delivery/read cursors, and cross-device sync.

### Scale
- **DAU / peak concurrent connections?** **Messages/day and peak/sec?** Avg group size?
  *Why:* the connection fleet is sized by **concurrent connections**, not RPS; fan-out multiplies message writes.

### Consistency / delivery guarantee
- **Ordering guarantee?** Per-conversation ordering (the norm) or global?
  *Why:* per-conversation ordering via a sequence number is achievable; global ordering is neither needed nor cheap.
- **Delivery guarantee?** Can a message ever be lost? Duplicated?
  *Why:* answer is **at-least-once + idempotent (dedup)** → the *effect* of exactly-once. Never silently drop.

### Latency
- **Target message delivery latency?** (typically **< ~200 ms** end to end when both online.)
  *Why:* justifies the persistent-connection tier over polling.

### Security / compliance
- **End-to-end encryption required?** (WhatsApp/Signal: yes; Slack: no.)
  *Why:* E2EE removes the server's ability to read messages → **no server-side search, no server fan-out of
  plaintext, key management per device.** A first-class architectural constraint, so ask early.

---

## 3. Functional requirements

### Must have
1. **Send/receive messages in real time** — 1:1 and group, sub-second when both parties are online.
2. **Durable history** — messages persist; a user can load past conversations and **sync on any device**.
3. **Delivery semantics** — **ordered per conversation**, **no loss**, **no duplicates** (at-least-once + dedup).
4. **Presence** — show online/offline (and last-seen), plus "typing…".
5. **Delivery & read receipts** — sent → delivered → read.
6. **Offline delivery** — persist while offline; **push notification**; deliver missed messages on reconnect.

### Nice to have (name, then defer)
- **Group chat at large scale** (10k+ / broadcast) — call out fan-out amplification; design the seam.
- **Media/attachments** — object-store upload + URL in the message; note it, defer detail.
- **End-to-end encryption** — mention as a mode; note what it removes (server search/fan-out of plaintext).
- **Search** — server-side index (incompatible with E2EE); defer.
- **Reactions, edits, deletes, threads** — message mutations; mention, defer.

> **Interview line:** *"Must-haves: real-time 1:1 + group messaging, durable multi-device history, ordered
> at-least-once delivery with dedup, presence, receipts, and offline push. I'll design the seams for large-group
> fan-out, media, and E2EE, and defer search and message edits unless we have time — and I'd confirm group size
> and whether E2EE is required up front, because both reshape the design."*

---

## 4. Non-functional requirements (quantified)

| NFR | Target | Why |
|---|---|---|
| **Delivery latency** | **p99 < ~200 ms** end-to-end when both online | It has to *feel* instant → persistent connections. |
| **Concurrent connections** | **hundreds of millions** live sockets | Sizes the gateway fleet (the dominant cost). |
| **Throughput** | **tens of billions of messages/day**; large group fan-out | Write + routing scale; fan-out amplification. |
| **Ordering** | **monotonic per conversation** | Users must see messages in the order sent. |
| **Durability / delivery** | **at-least-once, no silent loss; dedup → no dup** | Losing a message is unacceptable; the store is truth. |
| **Availability** | **always accept + eventually deliver**; degrade receipts/presence first | A dropped connection must never drop a message. |
| **Multi-device consistency** | all devices converge on the same history & read state | One account, many devices is the norm. |
| **Presence accuracy** | **best-effort / eventually consistent** | Cheap and self-healing; not worth strong consistency. |

**Dominant NFRs:** **massive concurrent-connection scale** + **low-latency ordered delivery** + **no message loss
(durable store, at-least-once + dedup)** + **multi-device sync**. Presence and receipts are explicitly allowed to
be softer.

---

## 5. What we are explicitly NOT building (this pass)

- The **mobile/web clients** themselves — we own the server side and the wire protocol.
- **Media storage internals** — reuse an object store + CDN; the message just carries a URL (seam to a file-storage
  system, *coming later*).
- **The push providers** (APNs/FCM) — reuse the [notification system](../notification-system/README.md) for
  offline delivery; we own the trigger, not the provider integration.
- **Full E2EE key-management protocol** (Signal/Double-Ratchet) — note it as a mode and its architectural impact,
  don't implement the crypto.

→ Next: **Capacity** *(coming)*
