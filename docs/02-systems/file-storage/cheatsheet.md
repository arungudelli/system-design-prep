# File Storage & Sync — Cheatsheet

> One-page revision. Reproduce from memory and you can drive the interview.

---

## Problem
Dropbox/Google Drive: store files, sync across a user's devices in seconds, share, version — efficient over the network,
never lose a byte, at 100Ms of users and exabytes.

## Profile (say first)
**Metadata/data split + bandwidth efficiency.** Blob storage is commodity (object store); the design is a **consistent
sharded metadata service** + **chunking/dedup/delta sync** + a **notification loop**. Read-heavy, storage-dominated.

## Requirements
- **Must:** chunked resumable up/download, near-real-time multi-device sync, delta transfer, dedup, durable + consistent
  file tree, versioning, conflict handling.
- **Defer/seams:** sharing (namespaces), search/previews, E2EE (kills cross-user dedup), selective sync.
- **Dominant NFRs:** durability (11 nines) · metadata strong consistency · bandwidth/storage efficiency · sync latency (sec).

## Numbers (example)
| | |
|---|---|
| Storage | 500M × 50 GB ≈ 25 EB raw → ~12–17 EB deduped (+EC overhead) |
| Dedup | store unique chunks once → biggest cost lever; free uploads if content exists |
| Upload BW | ~100 GB/s avg ingress → **delta sync + dedup** |
| Download BW | ~10× uploads → **CDN** immutable blocks |
| Metadata QPS | ~10^5–10^6 ops/s, **consistent** → sharded + cached (the real DB problem) |
| Chunks | ~10^12+ immutable blocks → refcount + GC |

## Architecture
```text
Client sync engine: watch FS · chunk+hash · reconcile via cursor
   │ 1.status(hashes) 2.PUT blocks 3.commit(meta) 4.GET /sync
   ▼                      ▼               ▼              ▲
[Block Service]     [Block Store]   [Metadata Svc]      │
 status/GC/refcnt → object store,    file tree,          │
   │                content-addr     versions,           │
  CDN (immutable)   (S3-class)       chunk lists, ACLs,   │
  download                           cursor (sharded/ns,  │
                                     strong consistency)  │
                                          │ on commit     │
                                          ▼               │
                                   [Notification Svc] ────┘  push "changed"→ other devices → pull
```

## Key ideas
- **Metadata/data split** — small transactional consistent tree vs huge immutable content-addressed bytes. *The* decision.
- **Chunk (~4 MB) + content-address (hash)** → delta sync + dedup + resumable + parallel, all from one move.
- **Client works in hashes:** `status()` → upload only missing → **commit = atomic transaction** (the file now exists).
- **Delta sync:** edit 1 MB of 1 GB → move one chunk. **Dedup:** identical content stored once (global, with PoP caveat).
- **Consistency:** metadata **strong** (single-shard commit); blocks **eventual** (immutable → lag is safe).
- **Sync:** notification **signal** ("namespace changed") + cursor **`/sync`** pull; poll is the correctness backstop.
- **Conflicts:** baseVersion detects divergence → **keep both** (conflicted copy), never silent loss.
- **GC:** refcount + conservative mark-and-sweep (delete late).

## Chunking
Fixed-size (simple; boundary-shift breaks dedup on edits) vs **content-defined / rolling hash** (insert stays local).
Client-side chunk+hash; server **verifies** hash(bytes).

## Bottlenecks (ranked)
1. **Metadata (QPS + consistency)** → shard by namespace, cache, single-shard commits
2. **Upload BW** → delta sync + dedup + resumable
3. **Download BW** → CDN immutable blocks + client chunk cache
4. Notification fan-out (shared folders) → signal + pull
5. Block GC → refcount + async sweep

## Failure one-liners
- Upload interrupted → resumable (status → upload missing); nothing visible pre-commit.
- Commit fails → idempotent retry; orphan blocks GC'd.
- Conflict → baseVersion → keep both. Corruption → hash check → refetch/EC repair.
- Metadata shard down → namespace-isolated; read-only/delayed sync. Notification missed → sync poll backstop.
- Dup upload → content-addressed no-op. Premature GC → conservative grace period.

## Patterns used
[sharding](../../01-patterns/sharding.md) · [caching](../../01-patterns/caching.md) · [idempotency](../../01-patterns/idempotency.md) ·
[queues+workers](../../01-patterns/queues-workers.md) · [real-time delivery](../../01-patterns/websockets-realtime.md) (notify) ·
[rate limiting](../../01-patterns/rate-limiting.md) (quotas).

## SQL vs NoSQL
Metadata → **sharded strongly-consistent transactional** (relational/NewSQL) by namespace. Block index (hash→size/refcount)
→ KV. Blocks → object store (content-addressed, CDN). Cursor → in metadata (monotonic per namespace).

## Summary line
> "Split metadata from data: a sharded, strongly-consistent metadata store (file tree, versions, ordered chunk-hash list,
> cursor) where the commit is an atomic single-shard transaction, plus immutable ~4 MB content-addressed chunks in an
> S3-class object store, CDN-fronted. Client chunks + hashes locally, uploads only missing chunks, then commits — giving
> delta sync + global dedup. Devices sync via a cursor with a notification push and a poll backstop. Conflicts keep both
> copies; chunks are refcounted and GC'd conservatively."

← Back to **[README](README.md)**
