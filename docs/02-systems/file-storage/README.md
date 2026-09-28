# File Storage & Sync (Dropbox / Google Drive)

> Design a file storage and **sync** service — a user drops a file in a folder on one device and it appears, byte-for-byte,
> on **all their other devices** and the web, within seconds; edits sync as **small deltas**, not whole-file re-uploads;
> storage is **deduplicated** so the same file stored by a million users costs (almost) one copy; and it all survives
> flaky networks, partial uploads, and two devices editing the same file at once — at **exabytes** of data and
> **hundreds of millions of users**.

The naive version ("POST the whole file to a server, GET it back") misses the entire interview. The hard parts are
**not** storing bytes — object stores do that. They are: **splitting files into chunks so you only move what changed**,
**content-addressed deduplication**, a **metadata service** that is the source of truth for the file tree (separate from
the bytes), **efficiently telling every other device that something changed**, and **resolving conflicts** when two
devices edit offline.

---

## Why it's a great system to study

- **It forces the metadata/data split** — the **file tree + versions + chunk lists** (small, transactional, strongly
  consistent) live in a totally different store from the **file bytes** (huge, immutable, eventually consistent). Seeing
  that split is half the interview.
- **Chunking + dedup is a beautiful lever** — split a file into content-addressed blocks and suddenly you get
  **delta sync** (upload only changed chunks), **dedup** (store each unique chunk once), and **resumable uploads** for free.
- **Sync is a distributed-systems problem** — detect local changes, reconcile against the server, pull remote changes,
  and handle the case where **both sides changed** — all over an unreliable network.
- **It composes patterns** — [sharding](../../01-patterns/sharding.md) the metadata DB, [caching](../../01-patterns/caching.md)
  hot metadata, [queues](../../01-patterns/queues-workers.md) for async processing, [idempotency](../../01-patterns/idempotency.md)
  for safe retries, and a [real-time notification channel](../../01-patterns/websockets-realtime.md) so devices learn of
  changes without polling.

---

## Read in this order

1. **[Requirements](requirements.md)** — problem, clarifying questions, functional + NFRs
2. **[Capacity](capacity.md)** — storage volume, bandwidth, dedup savings, metadata QPS
3. **[API & Data Model](api-data-model.md)** — chunked upload/download, sync cursor, metadata vs block schema
4. **[Architecture](architecture.md)** — sync client → metadata service + block service + notification service
5. **[Deep Dives](deep-dives.md)** — chunking, content-addressed dedup, delta sync, conflicts, notification, consistency
6. **[Trade-offs](tradeoffs.md)** — the decisions, both sides
7. **[Failure Scenarios](failure-scenarios.md)** — partial upload, conflict, metadata/block divergence, notification lag
8. **[Interview Questions](interview-questions.md)** — attempt first, then reveal
9. **[Cheatsheet](cheatsheet.md)** — 1-page revision
10. **[★ Principal Deep Dive](principal-deep-dive.md)** — the **staged stops** (single blob store → global) + max-scale variant
    (sharded metadata, cross-region, cold tiering, client-side E2E encryption) with a reconciliation table

---

## The 30-second version (know this cold)

- **Two stores, split on purpose.** A **metadata service** (file tree, versions, per-file **chunk list**, permissions)
  — small, transactional, **strongly consistent**, sharded by user/namespace — and a **block store** (the actual file
  bytes as immutable chunks) — huge, **content-addressed**, on object storage. **Metadata is truth; blocks are content.**
- **Files are chunked.** Split each file into blocks (e.g. **~4 MB**), hash each block. A file = an **ordered list of
  chunk hashes** in metadata. This one decision unlocks everything below.
- **Dedup is content-addressed.** A chunk's address **is** its hash → the same bytes are stored **once** globally
  (block-level dedup). Uploading a file you already have = upload nothing, just reference the hashes.
- **Sync is delta.** On change, the client hashes chunks, asks the server **which chunks are missing**, uploads only
  those, then commits new metadata. Editing 1 MB of a 1 GB file moves ~one chunk, not a gigabyte.
- **Devices learn via a notification channel**, not polling. A lightweight **notification service** tells a user's other
  devices "your namespace changed, pull"; the client then does a **cursor-based metadata sync** to get the delta.
- **Conflicts are inevitable.** Two devices edit the same file offline → the server keeps **both** (a "conflicted copy")
  rather than silently losing one. Metadata versioning makes this detectable.
- **Upload path:** client chunks + hashes → `has-chunks?` → upload missing chunks to block store → **commit metadata**
  (the atomic "the file now exists at version N") → notify other devices.
- **Bottlenecks:** **metadata QPS/consistency**, **upload bandwidth**, and **the notification fan-out** to a user's
  devices — *not* raw blob storage, which object stores handle.
