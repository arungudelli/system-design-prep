# File Storage & Sync — Deep Dives

The interview lives here: **chunking**, **content-addressed dedup**, **delta sync**, **conflict resolution**, the
**notification/sync loop**, **consistency (metadata vs blocks)**, and **garbage collection** — plus **sharing** and
**E2EE** as named seams.

---

## 1. Chunking (the decision that unlocks everything)

Split each file into blocks (e.g. **~4 MB**) and treat a file as an **ordered list of chunk hashes**. That one move buys:

- **Delta sync** — edit part of a file → only the changed chunk(s) move.
- **Dedup** — identical chunks (across files/users) stored once (see #2).
- **Resumable uploads** — upload chunk-by-chunk; resume from what's missing.
- **Parallelism** — upload/download chunks concurrently.

**Fixed-size vs content-defined chunking:**
- **Fixed-size (e.g. every 4 MB):** simple, but **inserting one byte at the front shifts every boundary** → every chunk
  hash changes → you re-upload the whole file (the "boundary-shift" problem).
- **Content-defined chunking (CDC, e.g. rolling hash / Rabin fingerprint):** boundaries are chosen by content, so an
  insert only changes the local chunk → **stable dedup across edits**. More complex, but the right answer for files that
  get edited in place.

> **Interview line:** *"I chunk files into ~4 MB content-addressed blocks, so a file is just an ordered list of hashes.
> That gives me delta sync, dedup, and resumable uploads in one move. If files get edited in the middle, I'd use
> content-defined chunking with a rolling hash so a single insert doesn't shift every boundary and force a full
> re-upload."*

---

## 2. Content-addressed deduplication

A chunk's **address is its hash** → identical content is physically stored once.

- **Levels:** **block-level** dedup (chunks) is the standard; whole-file dedup is a weaker special case. Block-level
  catches partial overlaps (two versions of a doc share most chunks).
- **Scope:** **cross-user (global) dedup** maximizes savings (everyone's copy of the same installer = one block) but has a
  **security caveat**: a "does this hash exist?" API can leak whether *someone* has a file (an attacker who has a file can
  confirm it's stored). Mitigate by **not revealing existence across trust boundaries** (e.g. per-user dedup, or requiring
  proof-of-possession). **Per-user dedup** is safer but saves less.
- **Hash choice:** a strong hash (SHA-256) makes collisions negligible; the hash is both the **integrity check**
  (verify on download) and the **address**.
- **Refcounting:** each chunk tracks how many versions reference it → drives GC (#7).

> **Interview line:** *"Blocks are addressed by their content hash, so identical content is stored once. Global dedup
> saves the most but can leak file existence, so across trust boundaries I'd scope dedup per-user or require
> proof-of-possession. The hash doubles as an integrity check on download."*

---

## 3. Delta sync (move only what changed)

The end-to-end bandwidth win, combining #1 and #2:

```text
1. Client edits report.pdf → re-chunks → [h1, h2', h3, h4]   (only h2 changed to h2')
2. status([h1,h2',h3,h4]) → missing:[h2']                    (server already has h1,h3,h4)
3. Upload only h2'  →  commit chunk list [h1,h2',h3,h4]
```
Editing 1 MB of a 1 GB file transfers **one chunk**, not a gigabyte. The download side is symmetric: a device pulls only
the chunks whose hashes it doesn't already have locally. Combined with a **local chunk cache**, most syncs move almost
nothing.

> **Interview line:** *"Delta sync falls out of chunking + content addressing: re-chunk, ask the server which hashes are
> missing, upload only those, commit the new hash list. A one-line edit to a huge file moves a single 4 MB chunk."*

---

## 4. Conflict resolution (never silently lose data)

Two devices edit the same file while offline, then both sync. You **must not** silently overwrite one.

- **Detect** via **version vectors / `baseVersion`:** each commit names the version it edited from. If the server's
  current version differs, the edits **diverged** → conflict.
- **Resolve — the Dropbox way: keep both.** Accept the first commit; for the second, create a **"conflicted copy"**
  (`report (conflicted copy, Bob's laptop).pdf`) and sync both everywhere. Human resolves. **No automatic merge** of
  binary files (you can't merge two JPEGs).
- **Last-writer-wins** is simpler but **loses data** — only acceptable where the product says so.
- **Text/structured files** *can* auto-merge (like Google Docs via **OT/CRDT**) — but that's a real-time collaborative
  editor, a different (harder) system; general file sync keeps both copies.
- **Metadata operations** (rename, move, delete vs edit) also conflict — resolve with the same version-based detection and
  conservative "keep both / tombstone" rules.

> **Interview line:** *"I detect conflicts with a baseVersion on every commit — if the server moved on, the edits
> diverged. For general files I keep both as a conflicted copy rather than auto-merging or overwriting, because you can't
> merge arbitrary binaries and silent data loss is unacceptable. Real-time text co-editing is a separate CRDT/OT problem."*

---

## 5. The notification + sync loop (near-real-time, cheaply)

How a change on one device reaches the others in **seconds** without everyone polling constantly:

- **Long-poll / WebSocket** per device to the **notification service**. On a commit, the server pushes a **signal** —
  "namespace X changed" — **not the data**.
- The device then does a **cursor-based `/sync`** to fetch the actual delta (changed file metadata), then downloads
  missing chunks. **Cheap push + efficient pull.**
- **Cursor is the source of truth for "what have I seen":** a monotonic per-namespace version. Miss a notification? The
  **periodic `/sync` poll** (every N seconds) is the backstop — notifications are an **optimization for latency**, not a
  correctness dependency.
- **Shared folders** turn this into a **fan-out**: a commit notifies **all members'** devices — reusing
  [real-time fan-out routing](../../01-patterns/websockets-realtime.md#fan-out-routing-getting-a-message-to-the-right-server).

> **Interview line:** *"Devices hold a long-poll to a notification service that pushes a tiny 'your namespace changed'
> signal; the device then pulls the delta via a cursor-based sync. The notification is just a latency optimization — a
> periodic sync poll is the correctness backstop, so a missed push never means a missed change."*

---

## 6. Consistency: metadata strong, blocks eventual

The deliberate split that makes the whole thing work:

- **Metadata = strongly consistent.** The **commit is a transaction**: check baseVersion, write the new version + chunk
  list, bump refcounts, advance the cursor — atomically, on a **single shard** (namespace-sharded so it *is* single-shard).
  You must never see half a commit, a file pointing at a not-yet-committed chunk, or a phantom tree.
- **Blocks = eventually consistent.** Chunks are **immutable + content-addressed**, so propagation lag / replication delay
  is harmless — a hash always maps to the same bytes, and you can verify on read. Metadata may briefly reference a chunk a
  given replica hasn't seen; the client just retries / fetches from origin.
- **Ordering:** commits within a namespace are **serialized** by the metadata transaction → a total order per namespace,
  which is what the cursor exposes. (Same idea as chat's [per-conversation sequence](../chat-system/deep-dives.md#2-ordering--deduplication).)

> **Interview line:** *"Metadata is strongly consistent — the commit is a single-shard transaction, so you never see half
> a file tree. Blocks are eventually consistent, which is safe because they're immutable and content-addressed: a hash
> always means the same bytes. That split lets me get transactional correctness where it matters and cheap object-store
> scale where it doesn't."*

---

## 7. Garbage collection (trillions of immutable chunks)

Immutable, deduped chunks can't be deleted when a file is — others may reference them.

- **Reference counting:** each chunk's refcount ++ on commit, -- when a referencing version is deleted/expired. Refcount 0
  ⇒ candidate for deletion.
- **Safe deletion via mark-and-sweep**, async and off the hot path, with a grace period — because a concurrent upload
  might reference a chunk that momentarily hit 0. Deleting a chunk that's still referenced = data corruption, so GC is
  **conservative** (delete late rather than early).
- **Version retention** feeds GC: expiring old versions is what eventually drops refcounts.

---

## 8. Sharing & permissions (the namespace seam)

- Model a shared folder as its own **namespace** with an **ACL** (owner/editor/viewer). Adding a user grants their client
  the namespace in its `/sync` set.
- Commits to a shared namespace **notify all members** (the fan-out in #5). Permission checks happen at the metadata
  service on every commit/read.
- **Public links** = a capability token granting read to a namespace/file without an account.

---

## 9. End-to-end encryption (a mode, and what it removes)

- **What it is:** the client encrypts chunks with keys the server never sees; the server stores **ciphertext**.
- **What it removes:** **cross-user dedup** (the same file encrypts to different ciphertext per user → no shared blocks),
  **server-side previews/thumbnails/search/transcoding** (server can't read content), and complicates **sharing** (key
  exchange per recipient).
- **What still works:** chunking, per-user delta sync, versioning, conflict detection (all operate on ciphertext +
  metadata). Convergent encryption can restore *some* dedup but weakens the security model.
- **Why ask early:** it trades a large chunk of the storage/bandwidth savings and all server-side processing for privacy —
  a first-class architectural decision, not a toggle.

→ Next: **[Trade-offs](tradeoffs.md)**
