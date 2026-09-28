# Video Streaming — Architecture

## 9. Architecture

Two independent systems glued by the object store: a **write path** (upload → async transcode pipeline → packaged
segments) and a **read path** (manifest + segments served from the **CDN**, with metadata/view services alongside).

```text
WRITE PATH (async, minutes)
  Uploader ──resumable chunks──▶ [ Upload Service ] ──▶ [ Object Store ]  (source master, durable)
                                       │ on complete: enqueue
                                       ▼
                                 [ Transcode Queue ]
                                       │
                            [ Transcode Worker Fleet ]   (elastic, spot; fan-out)
                             - split source into segments
                             - encode each segment × N renditions (in parallel)
                             - package HLS/DASH (segments + manifests)
                                       │ write outputs
                                       ▼
                                 [ Object Store ]  (segments, manifests, thumbnails — re-derivable)
                                       │ mark video ready
                                       ▼
                                 [ Metadata Service ]  status=ready, rendition list, manifest URL

READ PATH (real-time, sub-second)
  Player ──GET metadata/manifest──▶ [ Metadata Svc + cache ]         (watch page + manifest URL)
  Player ──GET manifest+segments──▶ [ CDN edge ] ──miss──▶ [ Regional cache ] ──miss──▶ [ Object Store origin ]
                    ▲  adaptive bitrate: player picks next segment's quality
  Player ──POST view/like──▶ [ Engagement Svc ] ──▶ [ view aggregation (stream) ] ──▶ approx counts
```

**Every stage justified:**
- **Upload service** — accepts **resumable chunked** uploads straight into the **object store**; on completion,
  **enqueues** the transcode job. The uploader never waits for processing.
- **Transcode queue + worker fleet** — the heavy async [fan-out](../../01-patterns/queues-workers.md): split into
  segments, encode each into **multiple renditions in parallel**, **package** HLS/DASH. **Elastic + spot** because it's
  bursty, embarrassingly parallel, and retryable. Writes outputs back to the object store.
- **Object store** — the seam between write and read: durable **source master** + re-derivable **segments/manifests**.
- **CDN (edge → regional → origin)** — serves ~**all** playback bytes from caches near the viewer; the object store
  origin only handles misses/fills. This is the whole delivery strategy (see [CDN](deep-dives.md#3-cdn-delivery-strategy)).
- **Metadata service** — read-heavy watch-page data + rendition list + manifest URL; **cached hard**, sharded by video.
- **Engagement service** — views/likes/comments; **views are aggregated approximately** (see
  [view counting](deep-dives.md#4-view-counting-at-scale)), never hot-row updates.

---

## 10. Request / data flow

### Upload → ready (the write path)
```text
1. Uploader → Upload Service: resumable chunks → source master in object store
2. On complete → status=uploaded → enqueue transcode job
3. Workers: split source into ~segments → for each (segment × rendition) encode in parallel
4. Package into HLS/DASH segments + media playlists + master manifest → write to object store
5. Metadata Service: write rendition list + manifest URL → status=ready
6. (Optionally) notify uploader; video is now playable
```
The video is "ready" only when **renditions + manifest exist** — partial transcode isn't visible.

### Playback (the read path)
```text
1. Player → Metadata Svc (cached): get video info + manifestUrl
2. Player → CDN: GET master manifest → learns available renditions
3. Player estimates bandwidth → picks a starting rendition (often low, for fast start)
4. Player → CDN: GET media playlist → GET segment files sequentially
5. Each segment: measure throughput/buffer → ADAPT next segment's quality (up/down)
6. Cache hit at edge = fast; miss → regional → origin (rare for hot content)
```

### View counting (approximate, high-volume)
```text
1. Player → Engagement Svc: view event (validated: real playback, deduped per user/session)
2. Event → stream/queue → aggregated (e.g. count in a fast store, or approximate structure)
3. Periodic flush → update approx view_count in metadata; display is eventually consistent
```

### Failure path
```text
Transcode fails a rendition → retry with backoff → DLQ + alert; source is safe (re-derive later). Video can go
   ready with the renditions that succeeded, or stay processing.
CDN edge miss → regional → origin; origin protected by cache + request coalescing.
Origin overload on a viral video → CDN absorbs it (cache once, serve millions); pre-warm edges for known spikes.
```

---

## 11. Bottlenecks (ranked)

| Rank | Bottleneck | Why | Detect via | Fix |
|---|---|---|---|---|
| 1 | **Egress / CDN delivery** | ~30+ Tbps; the cost + latency center | edge hit ratio, origin egress, rebuffer | **CDN-first**; maximize edge hit ratio; multi-CDN; per-title encoding to cut bits |
| 2 | **Transcode throughput** | huge bursty parallel compute | queue backlog, upload-to-ready lag | **elastic worker fleet**, segment-parallel, spot; prioritize |
| 3 | **Origin overload (cache miss storms)** | a viral/cold video hits origin | origin QPS spikes | **request coalescing**, tiered caches, pre-warm |
| 4 | **View-count writes** | 10^6/s to hot rows | counter latency | **aggregate/approximate**, stream + batch flush |
| 5 | **Metadata read QPS** | every watch page | metadata store QPS | **cache** hard; shard by video |

The signature problems: **serving egress from the edge** and **transcode throughput** — not storing bytes.

---

## 12. Scaling each bottleneck

- **Egress/CDN** → serve from **edge caches**; push **edge hit ratio** as high as possible (keep the hot head resident);
  **multi-CDN** for capacity + resilience + cost; **per-title / per-scene encoding** to send fewer bits for the same
  quality (a direct egress win). See [CDN](deep-dives.md#3-cdn-delivery-strategy).
- **Transcode** → **elastic worker fleet** on a [queue](../../01-patterns/queues-workers.md), autoscaled on backlog,
  running on **spot/preemptible** compute (async + retryable); **segment-level parallelism** so long videos don't
  serialize; priority lanes (short/popular creators first).
- **Origin protection** → **request coalescing** (collapse concurrent misses for the same segment into one origin fetch),
  **tiered caching** (edge → regional → origin), **pre-warming** for scheduled/known spikes (premieres).
- **View counts** → **aggregate** in a fast store / stream processor, **batch-flush** approximate totals; dedup real views
  ([idempotency](../../01-patterns/idempotency.md)). See [view counting](deep-dives.md#4-view-counting-at-scale).
- **Metadata** → **cache** aggressively (popular videos read millions of times), shard by `video_id`, read replicas.

---

## 20. Evolution with scale

| Stage | Shape | What forced the jump |
|---|---|---|
| **1 · Single rendition** | upload → transcode to one format → store → stream from a server | doesn't fit varied devices/networks; server can't serve egress; one CPU per video |
| **2 · Multi-rendition + async pipeline + CDN** | resumable upload → **queue + worker** transcode into N renditions → HLS/DASH → **CDN** delivery | device/network diversity → ABR; egress → CDN; heavy transcode → async fleet |
| **3 · CDN-first at scale + aggregated views + tiered caching (base target)** | edge/regional/origin tiers, request coalescing, elastic spot transcode, approximate view counting, cached sharded metadata | global viewers; viral spikes; view write volume; cost |
| **4 · Multi-CDN steering, per-title encoding, live, edge compute** | multi-CDN with steering, per-title/per-scene encoding, low-latency **live** pipeline, edge personalization | egress cost + resilience; encoding efficiency; live; scale |

> Transitions are driven by **device diversity (→ renditions + ABR)**, **egress (→ CDN-first, then multi-CDN)**, and
> **transcode load (→ elastic fleet)** — not by storing bytes.

→ Next: **[Deep Dives](deep-dives.md)**
