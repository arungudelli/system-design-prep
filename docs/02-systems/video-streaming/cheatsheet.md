# Video Streaming — Cheatsheet

> One-page revision. Reproduce from memory and you can drive the interview.

---

## Problem
YouTube/Netflix: upload videos → transcode into many renditions → stream smoothly to 100Ms of viewers worldwide with
fast start + adaptive quality; metadata, view counts, search. Billions of watch-hours/day, exabytes.

## Profile (say first)
**Read-heavy, CDN-first delivery** fed by a **heavy async transcode pipeline.** Write path and read path are different
systems joined by an object store. **Egress is the dominant cost.**

## Requirements
- **Must:** resumable upload, async transcode → multi-rendition, CDN adaptive-bitrate playback (sub-sec start), metadata,
  approx view counts, durable source.
- **Defer/seams:** search, recommendations (ML), comments, live, DRM.
- **Dominant NFRs:** egress/CDN delivery · smooth ABR playback · transcode throughput · source durability.
  (Exact counts + raw storage NOT the hard parts.)

## Numbers (example)
| | |
|---|---|
| Egress | tens of M concurrent streams × ~3 Mbps ≈ **~30+ Tbps** → CDN-first (the cost center) |
| Transcode | 500 hr/min uploaded × 6–10 renditions → huge elastic fan-out |
| Storage | multiple PB/day → exabytes; source durable, renditions re-derivable → tier |
| Views | ~10^5–10^6/s → aggregate/approximate (not hot-row UPDATE) |
| Cache hit | >90% for hot head → every 1% = huge $ |

## Architecture
```text
WRITE (async, minutes):
 uploader ─resumable→ [Upload Svc] → [Object Store] (source master)
                          │ enqueue
                    [Transcode Queue] → [Worker Fleet: split→encode segment×rendition→package HLS/DASH]
                          │ → [Object Store] (segments, manifests) → [Metadata: status=ready]
READ (real-time, sub-sec):
 player → [Metadata+cache] (manifest URL)
 player → [CDN edge] ─miss→ [regional] ─miss→ [origin/object store]   (ABR: pick quality per segment)
 player → [Engagement Svc] → view stream → approx counts
```

## Key ideas
- **Write ≠ read.** Async transcode pipeline vs CDN-first playback. Design separately.
- **Transcode = segment-parallel fan-out** on elastic **spot** workers behind a queue; idempotent per (video,seg,rendition).
- **Renditions ladder + ABR:** manifest lists renditions; **player** picks quality per ~2–6s segment by bandwidth/buffer;
  start low → ramp; downshift before stalling.
- **CDN-first:** immutable segments cache forever; tiered edge→regional→origin; **request coalescing** kills miss-storms;
  hot head at edge, long tail → origin. Multi-CDN steering at scale.
- **View counting:** events → stream → aggregate → periodic flush; **approximate**; dedup/validate; HLL for uniques.
  Exact only for monetization.
- **Durability:** source master = 11 nines; renditions **re-derivable** → cheaper tier / regenerate.
- **Per-title encoding** = fewer bits, same quality = direct egress win.

## HLS/DASH
Player fetches master manifest → media playlist → segment files. Immutable segments = perfect CDN objects. CMAF = share
segments across HLS+DASH.

## Bottlenecks (ranked)
1. **Egress/CDN** → CDN-first, edge hit ratio, multi-CDN, per-title encoding
2. **Transcode** → elastic spot fleet + queue, segment-parallel
3. **Origin overload** → request coalescing + tiered cache + pre-warm
4. View writes → aggregate/approximate off-path
5. Metadata reads → cache + shard by video

## Failure one-liners
- Upload drop → resumable. Transcode fail → retry→DLQ, re-derive from durable source.
- Viral → request coalescing (1 origin fetch) + edge serves millions. CDN out → multi-CDN steering.
- Bad network → ABR downshift (degrade quality, don't stall). Metadata down → cached/stale.
- View store lag → off playback path, approximate anyway. Bit rot → EC repair; rendition re-derivable.

## Patterns used
[queues+workers](../../01-patterns/queues-workers.md) (transcode) · [caching](../../01-patterns/caching.md)/CDN (delivery) ·
[sharding](../../01-patterns/sharding.md) (metadata) · [idempotency](../../01-patterns/idempotency.md) (jobs + view dedup) ·
chunked upload + object store (from [file storage](../file-storage/README.md)) · [rate limiting](../../01-patterns/rate-limiting.md) (quotas).

## SQL vs NoSQL
Metadata → NoSQL/sharded relational by video + **cache hard**. Transcode jobs → KV. View counts → fast counter/stream +
flush. Comments → wide-column by video. Blobs → object store + CDN (immutable segments).

## Summary line
> "Two systems joined by an object store: async segment-parallel transcode (elastic spot fleet → bitrate ladder → HLS/DASH)
> and CDN-first playback (player fetches a manifest, adaptively pulls immutable segments from the edge). Egress is the
> dominant cost, so maximize edge hit ratio, request-coalesce miss-storms, and use per-title encoding + multi-CDN. Source
> master is durable; renditions re-derivable. View counts are aggregated approximately, off the playback path."

← Back to **[README](README.md)**
