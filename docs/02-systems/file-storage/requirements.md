# File Storage & Sync — Requirements

## 1. Problem statement (the interview prompt)

> "Design a file storage and sync service like **Dropbox / Google Drive**. Users store files, access them from any
> device and the web, and any change on one device **syncs to all their other devices**. Files can be **shared** with
> other users. It must be efficient over the network, reliable (never lose or corrupt a file), and scale to hundreds of
> millions of users and exabytes of data."

Deliberately broad. The three answers that reshape everything: **how big can files get** (drives chunking), **how many
devices/how real-time must sync be** (drives the notification channel), and **whether it's end-to-end encrypted** (kills
server-side dedup across users). Clarify before drawing boxes.

---

## 2. Clarifying questions

### Product / scope
- **What operations?** upload, download, sync across devices, share (with users / public link), versioning/restore?
  *Why:* sync + sharing are the hard parts; a pure upload/download blob store is a different, easier problem.
- **Max file size?** a few MB, or multi-GB video files?
  *Why:* large files **force chunking + resumable uploads**; if everything is tiny, chunking is less critical.
- **How real-time must sync be?** seconds, or is "eventually" fine?
  *Why:* "within seconds" justifies a **push notification channel**; "eventually" could be periodic polling.
- **Edit granularity?** whole-file replace, or in-place edits of huge files?
  *Why:* in-place edits are what make **delta/block-level sync** pay off vs. re-uploading the file.
- **Versioning / restore?** keep history? for how long?
  *Why:* version history multiplies metadata and (un-deduped) storage; drives retention policy.
- **Sharing model?** per-file/folder ACLs, shared folders, public links?
  *Why:* shared folders complicate the "namespace" and the notification fan-out (notify all members).

### Scale
- **Users / DAU? Avg storage per user? Avg file size + count? Reads:writes ratio?**
  *Why:* sizes the block store, metadata DB, and bandwidth. File storage is **read-heavy** and **storage-dominated**.

### Consistency / durability
- **Durability target?** (essentially **never lose a file** — 11 nines, like S3.)
  *Why:* files are irreplaceable user data → durability is non-negotiable; drives replication/erasure coding.
- **Consistency of the file view?** must a device always see a consistent tree?
  *Why:* **metadata strongly consistent** (you must not see half a commit); **blocks eventually consistent** is fine.

### Security
- **End-to-end encryption?** (consumer Dropbox: no, server-side dedup; some competitors: yes.)
  *Why:* client-side E2EE means the server sees ciphertext → **cross-user dedup breaks** (identical files encrypt
  differently per user) and server-side preview/search dies. A first-class architectural constraint — ask early.

---

## 3. Functional requirements

### Must have
1. **Upload / download files** — including **large files** (chunked, resumable).
2. **Sync across a user's devices** — a change on one device propagates to all others (and web) within seconds.
3. **Delta sync** — transfer only what changed (changed chunks), not whole files.
4. **Deduplication** — identical content isn't stored/transferred repeatedly.
5. **Durable & consistent** — never lose or corrupt a file; devices see a consistent file tree.
6. **Versioning** — keep recent versions; restore a previous one.
7. **Conflict handling** — concurrent edits don't silently lose data.

### Nice to have (name, then defer)
- **Sharing** (shared folders, permissions, public links) — call out the namespace + notification impact; design the seam.
- **Search / previews / thumbnails** — server-side processing pipeline; incompatible with E2EE.
- **End-to-end encryption** — mention as a mode; note it removes cross-user dedup + server processing.
- **Offline edits** — clients queue changes and reconcile on reconnect (partly inherent to sync).
- **Selective sync / smart sync** — don't materialize every file on every device.

> **Interview line:** *"Must-haves: chunked resumable upload/download, near-real-time multi-device sync with delta
> transfer and dedup, durable storage with a strongly-consistent file tree, versioning, and conflict handling that
> never loses data. I'll design the seams for sharing and E2EE — noting E2EE kills cross-user dedup — and defer search/
> previews. I'd confirm max file size and how real-time sync must be, since both reshape the design."*

---

## 4. Non-functional requirements (quantified)

| NFR | Target | Why |
|---|---|---|
| **Durability** | **~11 nines** (never lose a file) | Irreplaceable user data → replication / erasure coding. |
| **Metadata consistency** | **strong** (read-your-writes on the file tree) | You must never see half a commit or a phantom file. |
| **Block consistency** | **eventual** is fine | A chunk is immutable + content-addressed; propagation lag is harmless. |
| **Sync latency** | **seconds** device-to-device when online | It must feel live → push notification, not slow polling. |
| **Availability** | **high**; degrade to read-only / delayed sync before losing data | Losing a byte is worse than being briefly slow. |
| **Storage efficiency** | **dedup + compression**; store unique chunks once | Exabyte scale → dedup is a direct cost lever. |
| **Bandwidth efficiency** | **delta sync**; upload only changed chunks | Network is the scarce resource for the client. |
| **Scale** | 100Ms of users, **exabytes**, read-heavy | Sizes block store, metadata shards, bandwidth. |

**Dominant NFRs:** **durability + metadata consistency + bandwidth/storage efficiency (chunking, dedup, delta sync) +
near-real-time sync.** Block storage itself is a solved problem (object store) — the design effort is metadata, sync,
and dedup.

---

## 5. What we are explicitly NOT building (this pass)

- **The object store internals** — reuse an S3-class store (replication, erasure coding, 11-nines durability); we design
  what goes *around* it (chunking, addressing, dedup), not the disk layer.
- **The desktop/mobile client internals** — we own the sync protocol + server side, not the OS filesystem watcher.
- **Full-text search / preview / transcoding pipelines** — mention as an async processing seam; don't build.
- **Full E2EE key management** — note it as a mode and its impact (no cross-user dedup, no server processing), don't
  implement the crypto.

→ Next: **[Capacity](capacity.md)**
