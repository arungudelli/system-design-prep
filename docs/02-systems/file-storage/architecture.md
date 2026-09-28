# File Storage & Sync — Architecture

## 9. Architecture

The system splits cleanly along the **metadata/data boundary**, with a **sync client** doing real work on the device and
a **notification service** closing the real-time loop.

```text
  Client sync engine (per device)
   - watch filesystem, chunk + hash changed files
   - reconcile local vs server via cursor
   - upload missing chunks / download new chunks
        │
        │ 1. status(hashes)   2. PUT blocks     3. commit(metadata)     4. GET /sync
        ▼                          ▼                    ▼                     ▲
  ┌───────────────┐        ┌──────────────┐     ┌──────────────────┐         │
  │ Block Service │        │  Block Store │     │ Metadata Service │         │
  │  status / GC  │───────▶│ (object store│     │  file tree,      │         │
  │  refcounting  │        │  S3-class,   │     │  versions,       │         │
  └───────────────┘        │  content-    │     │  chunk lists,    │         │
        ▲                   │  addressed)  │     │  ACLs, cursor    │         │
        │ CDN (immutable    └──────────────┘     │  (sharded by ns, │         │
        │ blocks by hash)                        │  strongly consistent)     │
   download path                                 └────────┬─────────┘         │
                                                          │ on commit:        │
                                                          ▼ "namespace changed"│
                                                 ┌──────────────────┐         │
                                                 │ Notification Svc │─────────┘
                                                 │ (long-poll / WS  │  push "pull!" to
                                                 │  per device)     │  user's OTHER devices
                                                 └──────────────────┘
```

**Every stage justified:**
- **Client sync engine** — the workhorse: watches the local filesystem, **chunks + hashes** changed files, computes what
  differs from the server, and drives the upload/commit/download protocol. Real intelligence lives on the client.
- **Metadata service** — the **source of truth** for the file tree, versions, per-file chunk lists, ACLs, and the sync
  cursor. **Strongly consistent, sharded by namespace**; the **commit** is its atomic transaction. The real DB problem.
- **Block service + block store** — `status` (which hashes are missing → dedup), block **PUT/GET**, and **refcount/GC**.
  Bytes live in an **S3-class content-addressed object store**; downloads are **CDN-fronted** (blocks are immutable).
- **Notification service** — holds a lightweight [long-poll/WebSocket](../../01-patterns/websockets-realtime.md) per
  device; on a commit it pushes a **signal** ("namespace X changed") to the user's *other* devices, which then **pull the
  delta** via `/sync`. Cheap push + efficient pull.

---

## 10. Request / data flow

### Upload / edit (the core write path)
```text
1. Client detects a changed file → splits into chunks → hashes each → [h1..h4]
2. Client → Block Service: status([h1..h4]) → missing:[h2,h4]      (dedup: h1,h3 already exist)
3. Client → PUT /blocks/h2, /blocks/h4  (idempotent, resumable; server verifies hash)
4. Client → Metadata: commit(path, chunks:[h1..h4], baseVersion:7)
      → metadata TX: check baseVersion (conflict?) → write version 8 → bump refcounts → advance cursor
      → 201 {version:8}         (this is the atomic "the file now exists at v8")
5. Metadata → Notification Service: "namespace changed"
6. Other devices receive the nudge → GET /sync?cursor → get {report.pdf v8, chunks} → download missing chunks → apply
```
**Persist blocks, then commit metadata, then notify** — until the commit, uploaded blocks are orphan bytes (GC'd later);
after it, the file is atomically visible.

### Download / new device
```text
1. Client → GET /sync?cursor=0 → full tree (or selective) as changes
2. For each file → GET metadata chunk list → GET /blocks/{hash} for chunks it lacks (CDN) → reassemble
```

### Conflict (two devices edit the same file offline)
```text
Device A commits report.pdf baseVersion:7 → becomes v8. ✔
Device B (also edited from v7) commits baseVersion:7 → server sees current=8 ≠ 7 → 409 Conflict
   → server keeps BOTH: creates "report (conflicted copy, Device B).pdf" → both sync everywhere. No data lost.
```

### Failure path
```text
Upload interrupted → re-run status() → upload only still-missing chunks (resumable). No commit yet = nothing visible.
Commit fails / times out → client retries (idempotent on (path, chunks, baseVersion)); orphan blocks GC'd by refcount.
Notification missed → client's periodic /sync poll catches up anyway (notification is an optimization, not truth).
```

---

## 11. Bottlenecks (ranked)

| Rank | Bottleneck | Why | Detect via | Fix |
|---|---|---|---|---|
| 1 | **Metadata service (QPS + consistency)** | high-QPS transactional commits + tree reads must be consistent | commit latency, shard hot spots | **shard by namespace**; cache hot reads; keep commits single-shard |
| 2 | **Upload bandwidth** | ingress of new/changed bytes | ingress rate, upload latency | **delta sync + dedup** (move only new chunks); resumable |
| 3 | **Download bandwidth** | read-heavy; popular files | egress, CDN hit rate | **CDN** immutable blocks; client-side chunk cache |
| 4 | **Notification fan-out** | shared folders → notify many devices | notify lag, connection count | lightweight signal + pull; reuse real-time patterns |
| 5 | **Block GC / refcounting** | trillions of chunks; safe deletion | orphan-block growth | async refcount + mark-and-sweep GC |

The signature problems: **a consistent, sharded metadata service** and **moving as few bytes as possible** — *not* the
blob storage, which the object store handles.

---

## 12. Scaling each bottleneck

- **Metadata** → **shard by `namespace_id`** so each user/shared-folder's tree + commits + sync stream are co-located and
  commits are single-shard transactions; **cache** hot tree reads ([caching](../../01-patterns/caching.md)); read replicas
  for listing-heavy load. This is the crux of the design.
- **Upload BW** → **delta sync** (chunk + `status` + upload-missing) and **dedup** mean you move only genuinely new
  content; **resumable** uploads survive flaky networks; compress chunks.
- **Download BW** → **CDN** the content-addressed (immutable) blocks — cache by hash forever, no invalidation; clients
  keep a local chunk cache and only fetch what they lack.
- **Notification** → the push channel carries a **signal**, not data; shared-folder fan-out reuses
  [real-time fan-out routing](../../01-patterns/websockets-realtime.md#fan-out-routing-getting-a-message-to-the-right-server);
  a periodic `/sync` poll is the backstop if a nudge is missed.
- **GC** → **reference counting** on chunks (++ on commit, -- on version expiry) + async **mark-and-sweep** to reclaim
  orphan blocks safely; never delete on the hot path.

---

## 20. Evolution with scale

| Stage | Shape | What forced the jump |
|---|---|---|
| **1 · Blob upload** | POST whole file to a server → local disk / one object store; download it back | no sync; re-uploads whole files; one box; no dedup |
| **2 · Metadata + block split + chunking** | separate metadata DB (file tree/versions) + object store for chunks; chunk + hash; **dedup**; resumable uploads | large files; wasted bandwidth; need history + dedup |
| **3 · Sync + notifications + delta (base target)** | cursor-based **/sync**, **notification service** for near-real-time multi-device, delta up/download, sharded metadata, CDN downloads | multi-device users; "it must feel live"; bandwidth cost |
| **4 · Global scale** | metadata sharded + geo-replicated, cross-region block storage, **cold tiering**, conflict handling at scale, selective/smart sync, optional client-side E2EE | exabytes, global users, storage cost, offline conflicts, privacy |

> Transitions are driven by **bandwidth waste (→ chunking/delta/dedup)**, **multi-device real-time (→ sync + notify)**,
> and **metadata QPS/consistency (→ sharding)** — not by the difficulty of storing bytes.

→ Next: **[Deep Dives](deep-dives.md)**
