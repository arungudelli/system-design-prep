# File Storage & Sync — API & Data Model

> The API splits along the **metadata/block boundary**: a **metadata/sync API** (small, transactional, consistent — the
> file tree, commits, and a **sync cursor**) and a **block API** (huge, content-addressed, immutable — upload/download
> chunks by hash). The whole protocol is built around one move: **the client works in chunk hashes, asks the server what
> it's missing, uploads only that, then commits metadata.**

---

## The upload protocol (chunk → check → upload → commit)

### 1. Client chunks & hashes locally
```text
Split file into ~4 MB blocks → [h1, h2, h3, …]   (hash = SHA-256 of chunk content)
```

### 2. Ask the server which chunks it's missing (dedup check)
```text
POST /v1/blocks/status
{ "hashes": ["h1","h2","h3","h4"] }
→ { "missing": ["h2","h4"] }        # server already has h1,h3 globally → don't upload them
```
This is where **dedup + delta sync happen**: only genuinely new content is uploaded. Re-uploading a file you (or anyone)
already stored ⇒ `missing: []` ⇒ upload nothing.

### 3. Upload the missing blocks (content-addressed, resumable)
```text
PUT /v1/blocks/{hash}       # body = chunk bytes; server verifies hash(body)==hash
→ 201 Created               # idempotent: PUT same hash again = no-op (immutable, content-addressed)
```
Blocks are **immutable and addressed by their hash** → uploads are naturally **idempotent** (safe to retry a failed
chunk) and **resumable** (re-run `status`, upload only what's still missing). See
[idempotency](../../01-patterns/idempotency.md).

### 4. Commit the file metadata (the atomic "it now exists")
```text
POST /v1/files/commit
{ "path": "/docs/report.pdf",
  "chunks": ["h1","h2","h3","h4"],   # ordered chunk list = the file
  "baseVersion": 7,                   # the version I edited from (for conflict detection)
  "size": ..., "mtime": ... }
→ 201 { "fileId": "f_9", "version": 8 }        # or 409 Conflict → conflict handling
```
The **commit is the transaction**: until it succeeds, uploaded blocks are just orphan bytes. `baseVersion` lets the
server detect **concurrent edits** (someone else committed version 8 first → conflict). This is the consistency anchor.

---

## The download protocol

```text
GET /v1/files?path=/docs/report.pdf     → { fileId, version, chunks:[h1..h4], size }
GET /v1/blocks/{hash}                    → chunk bytes  (served via CDN; cache by hash forever — immutable)
```
Client fetches the chunk list from metadata, then pulls **only the chunks it doesn't already have locally** (delta
download), reassembles in order. Because blocks are immutable + content-addressed, they're **infinitely CDN-cacheable**.

---

## The sync protocol (cursor-based — how devices catch up)

```text
GET /v1/sync?cursor=<opaque>            # "what changed in my namespaces since this cursor?"
→ { "changes": [ {path, fileId, version, chunks, deleted?}, … ],
    "cursor": "<new>" }                 # advance the cursor; persist it locally
```
- A **monotonic cursor per namespace** (like the sequence number in [chat](../chat-system/api-data-model.md#the-sync-protocol-after-reconnect)):
  the client says "I'm at cursor X," the server returns everything since. Reconnect after a week ⇒ one call returns the delta.
- The client is **told to sync** by the notification channel (below); it then calls `/sync` to get the actual changes.

### Notification channel (the "something changed, pull" nudge)
```text
Client holds a long-poll / WebSocket to the notification service.
← { "type": "namespace_changed", "namespaceId": "ns_1" }     # signal only, no data
Client → GET /v1/sync?cursor=…                                # then pulls the delta
```
Cheap signal + pull-the-delta keeps the push channel tiny and the transfer efficient. See
[real-time delivery](../../01-patterns/websockets-realtime.md).

---

## Data model

### Access patterns first
1. **List a folder / walk the tree** for a user → by namespace + path.
2. **Get a file's chunk list** (to download) → by file id.
3. **Commit a new version** (transactional; detect conflict via baseVersion) → the hot write.
4. **Sync since cursor** → changes in a namespace after a monotonic version.
5. **Block: does this hash exist? fetch it** → point lookup by hash.
6. **Refcount / GC** → how many files reference this chunk?

### Metadata store (small, transactional, **strongly consistent**, sharded by namespace)
```text
namespaces:  namespace_id (PK) | owner/type (user | shared_folder) | current_cursor (monotonic)
files:       file_id (PK) | namespace_id | path | current_version | deleted?
file_versions: file_id + version (PK) | ordered chunk_hash[] | size | mtime | author_device | created_at
                (cursor/seq within namespace → drives /sync)
acls:        namespace_id + principal (user/group) | permission (owner/editor/viewer)
```
- **Sharded by `namespace_id`** so a user's (or shared folder's) whole tree + sync stream lives together and commits are
  a **single-shard transaction**. Versions are append-only → cheap history + restore.
- The **chunk list is the file** — the file "content" in metadata is just an ordered array of hashes.

### Block store (huge, immutable, **content-addressed**, object storage)
```text
chunk:       hash (PK = address) → bytes            # on S3-class object store
chunk_index: hash (PK) | size | refcount            # refcount for GC; ++ on commit, -- on version delete
```
- Key **is** the content hash → identical content collapses to one object (global dedup). **Immutable** → cache forever,
  no invalidation. **Refcount** drives garbage collection: a chunk is deletable only when no version references it.

---

## SQL or NoSQL? (per store)

- **Metadata → strongly-consistent, sharded relational (or NewSQL)**. It needs **transactions** (commit = atomic version
  bump + chunk-list write + conflict check) and **read-your-writes**; sharded by `namespace_id`. This is the classic
  place people wrongly reach for eventual consistency — don't; the file tree must be consistent.
- **Block index (hash → size/refcount) → KV/wide-column**, huge but simple point lookups by hash; refcount updates.
- **Blocks → object storage** (S3-class): 11-nines durability, erasure coding, cheap, CDN-frontable.
- **Sync cursor → part of the metadata store** (monotonic per namespace).

> **Interview line:** *"I split metadata from data. Metadata — the file tree, versions, and per-file ordered list of
> chunk hashes — goes in a sharded, strongly-consistent transactional store keyed by namespace, because a commit must be
> atomic and you can never see half a file tree. The bytes go in a content-addressed object store where the key is the
> chunk's hash, which gives me global dedup and immutable, CDN-cacheable blocks for free. The client works in hashes:
> ask what's missing, upload only that, then commit metadata as the transaction."*

→ Next: **[Architecture](architecture.md)**
