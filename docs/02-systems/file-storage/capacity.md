# File Storage & Sync — Capacity Estimation

> See the method in **[Capacity estimation](../../00-framework/capacity-estimation.md)**. For file storage the key
> numbers are **total stored bytes** (before and **after dedup**), **upload/download bandwidth**, **metadata QPS** (the
> transactional hot path), and **chunk counts** (blocks are the real unit of work). The dominant insight: **raw blob
> storage is a solved, cheap commodity — the pressure is on metadata and bandwidth, and dedup + delta sync are the two
> levers that bend the curve.**

---

## Assumptions (state them)

- **500M registered users**, **100M DAU**.
- Avg **~50 GB stored per user** (mix of active + archived).
- Avg file **~1 MB**, but a long tail of **multi-GB** files (video). Chunk size **~4 MB**.
- **Read-heavy:** downloads/syncs ≫ uploads (~ **10:1** or more).
- **Dedup + compression** save a large fraction of raw bytes (identical files, shared folders, common installers).

---

## Total storage

```text
Raw (pre-dedup):  500M users × 50 GB ≈ 25 EB (exabytes)
Dedup + compression (say ~30–50% saved on average) → ~12–17 EB actually stored
+ replication / erasure coding overhead (~1.3–1.5×) on top
+ version history (recent versions retained) adds a fraction more
```

→ **So what?** This is **object-store territory** (S3-class): erasure coding + multi-AZ replication for **11-nines
durability**, cheap $/GB, and **cold tiering** for archival data. You do **not** build this disk layer — you build what
sits on top. **Dedup is a direct multi-exabyte cost lever**, which is *why* content-addressed chunking earns its place.

---

## Deduplication (why content-addressing pays)

```text
Block-level dedup: store each unique chunk (by hash) ONCE globally.
Examples that dedup hugely: the same PDF/installer/photo shared by millions,
files copied inside a shared folder, re-saved documents (most chunks unchanged).
```

→ **So what?** Dedup converts "500M copies of the same 100 MB app" into ~one stored copy + 500M metadata references.
It also makes **uploads free when the content already exists** (client sends hashes; server says "already have it").
Dedup is both a **storage** and a **bandwidth** win — the single highest-leverage design choice.

---

## Upload / download bandwidth

```text
Uploads: say 100M DAU × 100 MB new/changed data/day ≈ 10 PB/day ingress
         10 PB / 10^5 s ≈ ~100 GB/s average ingress  (peak several×)
Downloads/sync: ~10× uploads (read-heavy) → ~1 TB/s aggregate egress territory
```

→ **So what?** Bandwidth, not IOPS, is the scarce resource. Two mitigations do the heavy lifting: **delta sync** (only
changed **chunks** move — editing 1 MB of a 1 GB file transfers ~one 4 MB chunk, not 1 GB) and **dedup** (never upload
content the server already has). **CDN** the hot downloads. This is why "upload the whole file" is disqualifying at scale.

---

## Metadata QPS (the transactional hot path)

```text
Sync polling/notifications, tree listings, "which chunks do you have?", commits:
100M DAU, each device syncing metadata frequently → ~10^5–10^6 metadata ops/sec
Metadata is SMALL per item (file → chunk-hash list, versions, ACLs) but HIGH QPS and must be CONSISTENT.
```

→ **So what?** The **metadata service is the real database problem**: high QPS, **strongly consistent** (you can't show
half a commit), **sharded by user/namespace**, and **cached** for hot reads. Blob storage scales by throwing disks at
it; metadata scales only with careful sharding + caching. This is where the interview's DB design lives.

---

## Chunk counts (blocks are the unit of work)

```text
~12–17 EB stored ÷ 4 MB/chunk ≈ 10^12+ unique chunks
Each chunk: a hash (address) + a block-store object + a metadata reference count
```

→ **So what?** You're managing **trillions of immutable, content-addressed blocks**. That drives the **chunk-index /
reference-counting** design (when is a chunk safe to garbage-collect? when refcount hits 0) and argues for a **content-
addressed key space** (hash → object) so the same content naturally collapses to one key.

---

## Notification fan-out

```text
A change in a user's namespace must reach that user's other devices (avg ~3) within seconds.
Shared folder with N members → notify all N members' devices.
```

→ **So what?** A **lightweight notification channel** (long-poll / WebSocket) pushes "namespace X changed, pull" — it
carries a *signal*, not the data. Shared folders make this a **fan-out** (all members), reusing the
[real-time delivery](../../01-patterns/websockets-realtime.md) patterns. The heavy metadata/block transfer happens on the
subsequent pull, keeping the notification cheap.

---

## Summary — what the numbers told us

| Number | Value | Design consequence |
|---|---|---|
| Total storage | ~25 EB raw → ~12–17 EB deduped | Object store + erasure coding + **cold tiering**; **dedup is the cost lever** |
| Dedup | store unique chunks once | Content-addressed blocks; **free uploads** when content exists |
| Upload BW | ~100 GB/s avg ingress | **Delta sync** (changed chunks only) + dedup; resumable uploads |
| Download BW | ~10× uploads | Read-heavy → **CDN** hot files |
| Metadata QPS | ~10^5–10^6 ops/s, **consistent** | **Sharded + cached metadata service** — the real DB problem |
| Chunks | ~10^12+ immutable blocks | Chunk index + **reference counting** + GC |
| Notify | signal to N devices in seconds | Lightweight push channel; fan-out for shared folders |

The profile: **storage-dominated but blob-storage is commodity** — the design effort is a **consistent, sharded
metadata service**, **bandwidth efficiency via chunking + delta sync + dedup**, and a **cheap notification channel**.
Design for metadata consistency and bytes-on-the-wire, not for the disks.

→ Next: **[API & Data Model](api-data-model.md)**
