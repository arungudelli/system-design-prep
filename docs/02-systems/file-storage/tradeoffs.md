# File Storage & Sync — Trade-offs

> Dominant requirements: **durability + metadata consistency**, **bandwidth/storage efficiency** (chunking, dedup, delta
> sync), and **near-real-time multi-device sync** — with blob storage treated as commodity. Judge every trade against those.

---

## 1. Whole-file vs chunked transfer

- **Whole-file:** dead simple; but re-uploads a gigabyte to change a byte, no dedup, no resume, no parallelism.
- **Chunked:** enables delta sync, dedup, resumable + parallel transfer; adds chunk-index + reassembly complexity.

**Choose:** **chunked** at any real scale — it's the foundation for the three efficiency wins. Whole-file only for a toy.
See [chunking](deep-dives.md#1-chunking-the-decision-that-unlocks-everything).

---

## 2. Fixed-size vs content-defined chunking

- **Fixed-size:** trivial boundaries; but an insert near the front shifts every boundary → full re-upload (dedup breaks
  on edits).
- **Content-defined (rolling hash):** boundaries follow content, so inserts stay local → stable dedup across edits; more CPU/complexity.

**Choose:** **fixed-size if files are mostly replaced whole** (photos, videos); **content-defined if files are edited in
place** (documents, VM images). Name the trade; many designs start fixed and add CDC where it pays.

---

## 3. Metadata & data: split stores vs one store

- **One store for both:** simpler; but the file tree needs transactions + strong consistency while bytes need cheap
  exabyte scale — no single store is great at both.
- **Split:** transactional consistent metadata store + content-addressed object store for bytes — each optimized for its job.

**Choose:** **split.** The metadata/data boundary is the defining decision of this system. See
[architecture](architecture.md#9-architecture).

---

## 4. Metadata consistency: strong vs eventual

- **Strong:** you never see half a commit or a phantom file; read-your-writes; needs transactions + careful sharding.
- **Eventual:** cheaper/simpler to scale; but a user seeing an inconsistent file tree is a correctness bug, not a nuisance.

**Choose:** **strong for metadata** (single-shard transactional commit), **eventual for blocks** (immutable +
content-addressed makes lag harmless). See [consistency](deep-dives.md#6-consistency-metadata-strong-blocks-eventual).

---

## 5. Dedup scope: global (cross-user) vs per-user

- **Global:** maximum storage savings (one copy of a popular file for everyone); but a "does this hash exist?" path can
  **leak file existence** across users.
- **Per-user:** safer (no cross-user leakage); saves less; still gets intra-user dedup + delta sync.

**Choose:** **global with a safety mitigation** (proof-of-possession, or don't expose existence across trust boundaries)
when savings matter and content isn't sensitive; **per-user** when privacy/isolation dominates. See
[dedup](deep-dives.md#2-content-addressed-deduplication).

---

## 6. Sync trigger: notification push vs polling

- **Polling only:** simple, robust; but a tight poll wastes QPS at idle and a slow poll makes sync feel laggy.
- **Notification push + backstop poll:** near-real-time with a cheap signal; a slow poll remains the correctness backstop.

**Choose:** **push for latency + a periodic poll for correctness.** The notification is an optimization; the cursor-based
poll guarantees no change is ever missed. See [notification loop](deep-dives.md#5-the-notification--sync-loop-near-real-time-cheaply).

---

## 7. Conflict handling: keep-both vs last-writer-wins vs auto-merge

- **Keep-both (conflicted copy):** never loses data; user resolves; clutters the folder briefly.
- **Last-writer-wins:** simplest; **silently loses** the other edit — usually unacceptable.
- **Auto-merge (OT/CRDT):** great for text, impossible for arbitrary binaries; it's a different, harder system.

**Choose:** **keep-both** for general file sync (data safety first); **auto-merge only for structured/text co-editing** as
a separate feature. See [conflicts](deep-dives.md#4-conflict-resolution-never-silently-lose-data).

---

## 8. Where to chunk/hash: client-side vs server-side

- **Client-side:** enables dedup *before* upload (skip existing chunks) and E2EE; costs client CPU and trusts the client's
  hashing (server must verify).
- **Server-side:** server controls chunking; but you've already spent the bandwidth uploading the whole file.

**Choose:** **client-side chunking + hashing** (the whole point is to avoid uploading what already exists), with the
**server verifying** `hash(bytes)==claimed hash` to stay trustworthy.

---

## 9. Download path: direct from object store vs CDN

- **Direct:** simple; but hot files hammer the store and add latency for distant users.
- **CDN:** immutable content-addressed blocks are perfectly cacheable (cache by hash forever, no invalidation).

**Choose:** **CDN** for downloads — content addressing makes caching trivially correct. See
[scaling](architecture.md#12-scaling-each-bottleneck).

---

## 10. Storage tiering: all-hot vs hot/cold tiers

- **All-hot:** simple, uniform latency; expensive at exabyte scale where most data is rarely accessed.
- **Hot/cold tiering:** move cold/archival chunks to cheaper storage (slower retrieval); big cost win.

**Choose:** **tier by access pattern** at scale — most stored bytes are cold; keep recent/active hot, archive the rest.

---

## The one-paragraph summary (say this)

> *"I split metadata from data. The metadata service — file tree, versions, per-file ordered chunk-hash list, ACLs, and a
> per-namespace sync cursor — is a sharded, strongly-consistent transactional store; the commit is an atomic single-shard
> transaction, so you never see half a file tree. The bytes live as immutable ~4 MB content-addressed chunks in an
> S3-class object store, CDN-fronted for reads. The client chunks and hashes locally, asks which hashes are missing,
> uploads only those, then commits — giving me delta sync and global dedup. Devices stay current via a notification
> service that pushes a cheap 'namespace changed' signal, backed by a cursor-based sync poll for correctness. Concurrent
> offline edits are detected by baseVersion and resolved by keeping both copies — never silent loss. Blocks are
> reference-counted and GC'd conservatively."*

→ Next: **[Failure Scenarios](failure-scenarios.md)**
