# File Storage & Sync — Interview Questions

> Answer each **out loud first**, then expand and self-critique against the
> [critique lens](../../00-framework/staff-level-thinking.md#how-your-answer-gets-graded-the-critique-lens).

---

## Core

**Q1. What's the overall shape of this system?**
<details markdown="1"><summary>Model answer</summary>

Split on the **metadata/data boundary**. A **metadata service** (file tree, versions, per-file **ordered chunk-hash
list**, ACLs, sync cursor) — small, transactional, **strongly consistent**, sharded by namespace. A **block store** (file
bytes as immutable **content-addressed ~4 MB chunks**) — huge, on an S3-class object store, CDN-fronted. A **client sync
engine** that chunks/hashes locally and drives upload/commit/download. A **notification service** that pushes "namespace
changed" so devices pull the delta. See [architecture](architecture.md).
</details>

**Q2. Why chunk files, and how big should chunks be?**
<details markdown="1"><summary>Model answer</summary>

Chunking (a file = an ordered list of chunk hashes) unlocks **delta sync** (move only changed chunks), **dedup** (store
identical chunks once), **resumable uploads**, and **parallelism** — all at once. Chunk size (~4 MB) trades metadata
overhead (smaller = more hashes to track) against transfer granularity (larger = re-send more per edit). See
[chunking](deep-dives.md#1-chunking-the-decision-that-unlocks-everything).
</details>

**Q3. Walk the upload path.**
<details markdown="1"><summary>Model answer</summary>

Client splits the file into chunks and hashes them → calls `status([hashes])` and the server returns which are
**missing** (dedup) → client `PUT`s only the missing chunks (idempotent, resumable, content-verified) → client
`commit(path, chunks, baseVersion)` which is an **atomic metadata transaction** (conflict check → new version → refcount
bump → cursor advance) → metadata notifies the user's other devices → they `/sync` and download the new chunks. **Persist
blocks, then commit, then notify.** See [flow](architecture.md#10-request--data-flow).
</details>

**Q4. How does deduplication work, and what's the catch?**
<details markdown="1"><summary>Model answer</summary>

A chunk's **address is its content hash**, so identical content is physically stored once (block-level, ideally
cross-user). Uploads become free when content exists (`status` says "already have it"). The catch: a global "does this
hash exist?" path can **leak file existence** across users → mitigate with **proof-of-possession** or **per-user dedup**
across trust boundaries. The hash also serves as an **integrity check**. See [dedup](deep-dives.md#2-content-addressed-deduplication).
</details>

**Q5. How does delta sync move only what changed?**
<details markdown="1"><summary>Model answer</summary>

Re-chunk the edited file, `status()` the new hash list → the server reports only the changed chunk(s) as missing → upload
those → commit the new list. Editing 1 MB of a 1 GB file moves **one 4 MB chunk**. Downloads are symmetric (pull only
hashes you lack), and a local chunk cache means most syncs move almost nothing. For in-place edits, **content-defined
chunking** keeps a single insert from shifting all boundaries. See [delta sync](deep-dives.md#3-delta-sync-move-only-what-changed).
</details>

---

## Consistency & sync

**Q6. What's consistent, what's eventual, and why?**
<details markdown="1"><summary>Model answer</summary>

**Metadata is strongly consistent** — the commit is a **single-shard transaction** (namespace-sharded), so you never see
half a file tree, a phantom file, or a pointer to a not-yet-committed chunk. **Blocks are eventually consistent** —
they're immutable and content-addressed, so replication lag is harmless (a hash always means the same bytes, verifiable on
read). Strong where correctness needs it, cheap where it doesn't. See [consistency](deep-dives.md#6-consistency-metadata-strong-blocks-eventual).
</details>

**Q7. How do a user's devices learn about changes in near-real-time without constant polling?**
<details markdown="1"><summary>Model answer</summary>

Each device holds a **long-poll/WebSocket** to a notification service. On commit, the server pushes a **signal**
("namespace X changed"), not the data; the device then does a **cursor-based `/sync`** to fetch the delta and downloads
missing chunks. The notification is a **latency optimization**; a **periodic sync poll** is the correctness backstop, so a
missed push never means a missed change. See [notification loop](deep-dives.md#5-the-notification--sync-loop-near-real-time-cheaply).
</details>

**Q8. Two devices edit the same file offline. What happens?**
<details markdown="1"><summary>Model answer</summary>

Each commit carries the **baseVersion** it edited from. The first commit wins; the second's baseVersion no longer matches
the server's current version → **conflict**. The server **keeps both** — creating a "conflicted copy" — and syncs both
everywhere. No last-writer-wins, no auto-merge of binaries, no silent loss. Real-time text co-editing (CRDT/OT) is a
separate, harder system. See [conflicts](deep-dives.md#4-conflict-resolution-never-silently-lose-data).
</details>

---

## Scale & storage

**Q9. What's the first bottleneck as you scale, and how do you address it?**
<details markdown="1"><summary>Model answer</summary>

The **metadata service** (high-QPS transactional commits + consistent tree reads) and **bandwidth**. Address metadata by
**sharding by namespace** (commits stay single-shard) + caching hot reads + read replicas. Address bandwidth with
**delta sync + dedup** (move only new chunks) and **CDN** for downloads (immutable blocks). Raw blob storage isn't the
bottleneck — the object store handles it. See [bottlenecks](architecture.md#11-bottlenecks-ranked).
</details>

**Q10. How do you delete data when chunks are deduplicated and shared?**
<details markdown="1"><summary>Model answer</summary>

**Reference counting**: ++ on commit, -- when a referencing version expires. Refcount 0 ⇒ deletion candidate. Reclaim via
**async mark-and-sweep with a grace period**, never on the hot path, re-checking refcount at sweep time — because a
concurrent upload might reference a chunk that momentarily hit 0. **Delete late, not early** — premature deletion of a
still-referenced chunk is data loss. See [GC](deep-dives.md#7-garbage-collection-trillions-of-immutable-chunks).
</details>

**Q11. How do you keep exabytes of storage affordable?**
<details markdown="1"><summary>Model answer</summary>

**Dedup** (store unique chunks once — the biggest lever), **compression**, **erasure coding** (durability at lower
overhead than 3× replication), and **hot/cold tiering** (most stored bytes are rarely accessed → archive them to cheaper,
slower storage). Blob storage is commodity; these four bend the cost curve. See [capacity](capacity.md).
</details>

---

## Reliability & security

**Q12. How do you guarantee a file is never lost or corrupted?**
<details markdown="1"><summary>Model answer</summary>

**Durability:** object store with **erasure coding + multi-AZ/region replication** (~11 nines). **Integrity:** the chunk
**hash is verified on read** — corruption is detectable and repaired from a replica. **Atomicity:** the **commit** is the
unit of truth, so interrupted uploads are invisible orphans (GC'd), never partial files. **Conflicts** keep both copies.
See [failure scenarios](failure-scenarios.md).
</details>

**Q13. What if an upload is interrupted halfway?**
<details markdown="1"><summary>Model answer</summary>

Nothing is visible until the **commit**, so a half-upload is just orphan blocks. On reconnect the client re-runs
`status()` and uploads **only the still-missing chunks** (resumable — that's a direct benefit of content-addressed
chunking), then commits. No corruption, no re-uploading what already landed.
</details>

**Q14. What changes with end-to-end encryption?**
<details markdown="1"><summary>Model answer</summary>

The server stores **ciphertext** and never sees keys → **cross-user dedup breaks** (same file → different ciphertext per
user), and **server-side previews/search/transcoding** die. Chunking, per-user delta sync, versioning, and conflict
detection still work on ciphertext + metadata. It trades a big chunk of storage/bandwidth savings and all server
processing for privacy — a first-class decision to confirm early. See [E2EE](deep-dives.md#9-end-to-end-encryption-a-mode-and-what-it-removes).
</details>

---

## Staff-level curveballs

**Q15. Where does this reuse patterns you've studied?**
<details markdown="1"><summary>Model answer</summary>

**Sharding:** metadata by namespace. **Caching:** hot tree reads + client chunk cache. **Idempotency:** content-addressed
blocks + idempotent commits. **Queues/workers:** async processing (thumbnails, GC, indexing). **Real-time delivery:** the
notification channel + shared-folder fan-out. **Rate limiting:** per-user quotas. It's a composition of the pattern
library around a **metadata/data split**.
</details>

**Q16. How is this different from a chat system — both have devices syncing via a cursor?**
<details markdown="1"><summary>Model answer</summary>

Both use a **monotonic per-namespace/per-conversation cursor** and a **notification channel + pull** for multi-device
sync — genuinely the same idea. The difference is the payload: chat moves **small ordered messages** over held sockets
(connection-bound), while file storage moves **large deduplicated byte-chunks** and its hard problems are **chunking,
dedup, and metadata transactions**, not holding connections. Chat is real-time-plane-dominated; file storage is
metadata-and-bandwidth-dominated.
</details>

**Q17. What changes at 10× and 100×?**
<details markdown="1"><summary>Model answer</summary>

**10×:** more metadata shards, bigger CDN footprint, more aggressive tiering. **100×:** **geo-replicated metadata** +
**cross-region block storage** for global latency and residency, mature **cold tiering**, conflict handling at scale, and
**selective/smart sync** so devices don't materialize everything. The metadata/data split + chunking backbone holds;
pressure is on metadata scale-out, cross-region consistency, and storage cost. See [principal deep dive](principal-deep-dive.md).
</details>

**Q18. When would you redesign this system?**
<details markdown="1"><summary>Model answer</summary>

When assumptions break: **real-time collaborative editing** becomes primary (needs CRDT/OT — a different system than
file sync); **E2EE** is mandated (kills dedup + server processing → rethink the storage economics); the workload shifts to
**huge-file streaming** (video) where a CDN/streaming design matters more than sync; or **compliance/residency** forces
per-region data isolation. Absent those, the metadata/data + chunking design holds.
</details>

---

## Self-scoring

Grade against the [15-point critique lens](../../00-framework/staff-level-thinking.md#how-your-answer-gets-graded-the-critique-lens):
did you split metadata from data, chunk + content-address for delta sync and dedup, make metadata strongly consistent via
a single-shard commit while blocks stay eventual, sync via cursor + notification-with-poll-backstop, resolve conflicts by
keeping both, reference-count for GC, and treat durability (erasure coding + hash integrity) as non-negotiable? Note your
weakest area and drill it.

← Back to **[README](README.md)** · Next: **[Cheatsheet](cheatsheet.md)**
