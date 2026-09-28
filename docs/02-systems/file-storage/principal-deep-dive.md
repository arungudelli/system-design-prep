# File Storage & Sync — Principal-Level Deep Dive (The Stops)

> The **"never be blind"** companion to the base File Storage docs. The system as a **staircase of stops** — smallest
> defensible design first, each bigger stop with its **exact trigger** — ending in the **top-stop (max-scale /
> deliberately over-engineered)** architecture plus a **reconciliation table** for when each heavy component is overkill
> and should be **removed**.
>
> Read the base first: [Requirements](requirements.md) · [Architecture](architecture.md)
> · [Deep Dives](deep-dives.md) · [Trade-offs](tradeoffs.md).
>
> **Say this in an interview:** *"I'll start with whole-file blob upload and grow it one bottleneck at a time. I know the
> sharded-metadata, geo-replicated, cold-tiered, content-defined-chunking, E2EE version — but each piece only earns its
> place when a specific ceiling forces it."*

---

## The stops (climb one lever at a time)

| Stop | Scale target | Shape | **Trigger that forces the NEXT stop** |
|---|---|---|---|
| **1 · Blob upload/download** | tiny; single user, few files | POST whole file → one object store; GET it back; metadata in one DB | no sync; whole-file re-uploads waste bandwidth; no dedup/history |
| **2 · Chunk + metadata/data split** | moderate | **chunk + hash** files; metadata DB (tree, versions, chunk lists) separate from **content-addressed** object store; **dedup**; resumable uploads | large files; multi-device users want live sync; bandwidth cost |
| **3 · Sync + notify + delta + sharded metadata (base target)** | 100Ms of users | cursor-based **/sync**, **notification service** (push + poll backstop), delta up/download, **metadata sharded by namespace**, CDN downloads, refcount GC | exabytes; global latency; storage cost; offline conflicts; privacy demands |
| **4 · Global: geo-replicated metadata, cross-region blocks, tiering, CDC, E2EE (top stop)** | exabytes, global | metadata **geo-replicated** + region-homed, **cross-region block storage**, **hot/cold tiering**, **content-defined chunking**, scaled conflict handling, **selective/smart sync**, optional **client-side E2EE** | (ceiling; real-time co-editing or E2EE mandates become different systems) |

> Most interviews want **Stop 2–3**. Reach into Stop 4 only when the prompt says "global / exabytes / data residency /
> end-to-end encrypted / edited-in-place huge files." Name the trigger each time.

---

## Top-stop architecture (max-scale)

```text
  Client sync engine (content-defined chunking, local chunk cache, selective sync, optional E2EE)
        │ status / PUT blocks / commit / GET sync   (geo-routed to home region)
        ▼
  ┌──────────────────────────┐         ┌───────────────────────────────┐
  │ Metadata Service         │         │ Block Service + Block Store   │
  │  sharded by namespace,   │         │  content-addressed chunks,     │
  │  GEO-REPLICATED,          │◀──────▶│  cross-region replication /    │
  │  namespace HOMED to a     │  refcnt │  erasure coding, HOT/COLD      │
  │  region (commit authority)│         │  TIERING, CDN front            │
  │  strong consistency        │         └───────────────────────────────┘
  └───────────┬──────────────┘                        ▲
              │ on commit                              │ CDN (immutable blocks by hash)
              ▼                                   download path
      ┌──────────────────┐
      │ Notification Svc │  region-aware push "namespace changed" → user's devices + shared-folder members
      └──────────────────┘
  Async pipeline (off critical path): thumbnails/previews, search index, GC mark-and-sweep, virus scan  [disabled under E2EE]
  Per-user quotas + rate limits · audit/compliance · residency routing
```

---

## Deep dives that only matter at the top stop

### Geo-replicated / region-homed metadata (Stop 3 → 4 trigger: **global users + data residency + five-nines**)
- **Why:** a global user base wants low-latency commits near them; regulations (GDPR) require some data stay in-region; a
  single region is a SPOF.
- **Design:** **home each namespace to a region** that owns its commit authority (keeps commits single-shard, single-writer
  → preserves strong consistency + the monotonic cursor); replicate for read locality and DR; route by namespace home.
  Cross-region access round-trips to the home region for commits.
- **Over-engineering watch:** *only for global latency / residency / five-nines.* Single-region multi-AZ metadata serves
  most; **remove** geo-replication below that (it adds real consistency complexity).

### Cross-region block storage + hot/cold tiering (trigger: **exabytes + cost + global reads**)
- **Why:** most stored bytes are cold; keeping everything hot in one region is expensive and far from distant users.
- **Design:** replicate/erasure-code blocks across regions; **tier by access** (recent/active hot, archival to cheaper,
  slower cold storage with a retrieval delay); CDN for hot reads. Content addressing makes cross-region caching trivially
  correct (immutable).
- **Over-engineering watch:** *only at real exabyte scale / global reads.* A single-region object store + CDN is plenty
  early; **remove** tiering when everything fits hot cheaply.

### Content-defined chunking (trigger: **large files edited in place**)
- **Why:** fixed-size chunking suffers the **boundary-shift** problem — an insert re-chunks the whole file, destroying
  dedup + delta savings for edited documents/VM images.
- **Design:** rolling-hash (Rabin) boundaries so an edit only changes local chunks. See
  [chunking](deep-dives.md#1-chunking-the-decision-that-unlocks-everything).
- **Over-engineering watch:** *only when files are edited in the middle.* For write-once media (photos/video), fixed-size
  is simpler and just as good; **remove** CDC there.

### Async processing pipeline (trigger: **previews/search/scan at scale**)
- **Why:** thumbnails, previews, full-text search, transcoding, and virus scanning are expensive and must stay **off the
  upload critical path**.
- **Design:** on commit, enqueue jobs ([queues + workers](../../01-patterns/queues-workers.md)); workers process blocks and
  write derived data/indexes. Idempotent, retryable, DLQ'd.
- **Over-engineering watch:** *only when those features exist and matter.* A pure sync product doesn't need it; and it's
  **fundamentally incompatible with E2EE** (server can't read content) — **remove** it in the E2EE mode.

### Client-side E2E encryption (trigger: **privacy product / regulatory mandate**)
- **Why:** the server must not read user content.
- **Design:** client encrypts chunks; server stores ciphertext + metadata; per-recipient key exchange for sharing.
- **What it removes:** **cross-user dedup** (ciphertext differs per user), **all server-side processing** (previews/
  search/scan). Convergent encryption can restore *some* dedup at a security cost. A first-class trade. See
  [E2EE](deep-dives.md#9-end-to-end-encryption-a-mode-and-what-it-removes).
- **Over-engineering watch:** *only when privacy is required.* If you need server search/previews/compliance-scan, E2EE is
  the wrong mode — **remove** it and keep server-readable storage.

### Selective / smart sync (trigger: **devices can't hold everything**)
- **Why:** a user with 2 TB in the cloud and a 256 GB laptop can't materialize every file.
- **Design:** sync **metadata for the whole tree** but download **block content on demand** (placeholder files hydrate on
  open); let users pin/unpin folders.
- **Over-engineering watch:** *only when cloud storage > device storage.* Small footprints sync everything; **remove** it.

---

## Reconciliation table — what forces each heavy component (and when to remove it)

| Component | Base docs (Stop 2–3) | Top stop (Stop 4) | Trigger to ADOPT | When it's OVERKILL → remove |
|---|---|---|---|---|
| Metadata | sharded by namespace, single region multi-AZ | geo-replicated, region-homed commit authority | global latency / residency / five-nines | one geo, 99.99% |
| Block storage | single-region object store + CDN | cross-region + hot/cold tiering | exabytes + global reads + cost | fits hot cheaply in one region |
| Chunking | fixed-size ~4 MB | content-defined (rolling hash) | large files edited in place | write-once media |
| Processing | none / minimal | async previews/search/scan pipeline | those features required | pure sync product |
| Encryption | TLS + at-rest | client-side E2EE | privacy / regulatory mandate | need server search/previews |
| Device footprint | sync everything | selective / smart sync | cloud storage > device storage | small footprints |
| Dedup | global (with PoP) | per-tenant / convergent under E2EE | privacy / trust boundaries | trusted single-tenant |

> Lead with Stop 2–3 (metadata/data split + chunking + content-addressed dedup + delta sync + cursor sync with
> notification + sharded strongly-consistent metadata + refcount GC). When pushed: *"for global exabyte scale with data
> residency, in-place-edited files, and optional E2EE, here's Stop 4 — each piece maps to a ceiling, and below that
> ceiling I'd remove it."*

---

## The 60-second Principal summary

> *"The defining decision is splitting metadata from data. Metadata — the file tree, versions, per-file ordered chunk-hash
> list, ACLs, and a per-namespace sync cursor — lives in a sharded, strongly-consistent transactional store where the
> commit is an atomic single-shard transaction, so you never see half a file tree. The bytes live as immutable ~4 MB
> content-addressed chunks in an object store, CDN-fronted. The client chunks and hashes locally, asks which hashes are
> missing, uploads only those, then commits — that one protocol gives me delta sync, global dedup, and resumable uploads.
> Devices stay live via a notification push plus a cursor-sync poll backstop; conflicts keep both copies rather than lose
> data; chunks are reference-counted and GC'd conservatively. Pushed to global scale I climb one lever at a time:
> region-homed geo-replicated metadata for latency and residency, cross-region blocks with hot/cold tiering for cost,
> content-defined chunking when files are edited in place, an async pipeline for previews and search, selective sync when
> devices can't hold everything, and client-side E2EE when privacy is mandated — knowing E2EE kills dedup and all server
> processing. Each piece has a named trigger, and below it I'd remove it."*

← Back to **[File Storage overview](README.md)** · Base **[deep dives](deep-dives.md)** · **[cheatsheet](cheatsheet.md)**
