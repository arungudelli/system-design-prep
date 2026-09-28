# Video Streaming — Capacity Estimation

> See the method in **[Capacity estimation](../../00-framework/capacity-estimation.md)**. For video the key numbers are
> **egress bandwidth** (the dominant cost and the whole reason for a CDN), **transcode compute** (CPU-hours per uploaded
> hour), **storage** (source + all renditions), and **view/metadata QPS**. The dominant insight: **egress dwarfs
> everything — a video is watched thousands of times for every time it's uploaded, so the read side, served from the
> edge, is the entire cost center.**

---

## Assumptions (state them)

- **~500 hours of video uploaded per minute** (YouTube-scale UGC).
- **~1B watch-hours per day**; **read:write hugely lopsided** — each uploaded hour is watched thousands of times.
- Each upload → **~6–10 renditions** (144p→4K, multiple bitrates/codecs).
- Avg streamed bitrate **~3 Mbps** (blended across devices/renditions).
- Peak concurrent streams in the **tens of millions**.

---

## Egress bandwidth (the dominant cost — size this first)

```text
1B watch-hours/day ÷ 86,400 s ≈ ~11.6M "hours being watched" per second → but think in streams:
tens of millions of concurrent streams × ~3 Mbps
  = 10^7 streams × 3×10^6 bps ≈ 3×10^13 bps = ~30 Tbps sustained  (peak much higher)
```

→ **So what?** This is **why the system is a CDN, not a server farm.** ~30+ Tbps cannot come from your origin — it must
be served from **CDN edge caches near viewers**. Your origin serves a tiny fraction (cache misses / cold long-tail).
Egress is also the **dominant $ cost** of the whole business, which is why **cache hit ratio at the edge** and
**per-title encoding efficiency** (fewer bits for the same quality) are top-line levers, not micro-optimizations.

---

## Transcode compute

```text
500 hours uploaded/min → 500 × 60 = 30,000 "video-hours" uploaded per hour
Each video-hour → ~6–10 renditions, each rendition roughly ~1–several × real-time CPU to encode
→ tens of thousands of concurrent transcode "video-hours" of work → a LARGE elastic worker fleet
```

→ **So what?** Transcoding is a **massive, bursty, embarrassingly-parallel batch workload** → an **elastic worker fleet**
fed by a [queue](../../01-patterns/queues-workers.md), scaled on backlog, ideally on **cheap/spot compute** (it's
async + retryable, so interruptions are fine). **Split each video into segments** so renditions encode **in parallel**
(a 2-hour movie doesn't wait on one CPU). This is the write-path scaling story.

---

## Storage (source + renditions)

```text
Uploaded/day: 500 hr/min × 1440 min ≈ 720,000 hours/day
Source master (say ~4 GB/hr high-quality): 720k × 4 GB ≈ ~3 PB/day of SOURCE
Renditions (all together often ~1–2× the source): another few PB/day
→ multiple PB/day added; exabytes over time
```

→ **So what?** **Object store territory** (S3-class, erasure-coded, 11 nines) with **tiering**: the vast **long tail is
rarely watched** → move cold videos + rarely-used renditions to **cheaper cold storage**. **Source masters are durable
but re-derivable renditions can be cheaper-tier or even regenerated on demand.** Storage is big but commodity — not the
hard part.

---

## View & metadata QPS

```text
Views: billions of view events/day → ~10^5–10^6 view increments/sec (peak higher on viral spikes)
Metadata reads: every watch page + player fetches video metadata + manifest → very high read QPS
```

→ **So what?** **Metadata is read-heavy** → [cache](../../01-patterns/caching.md) aggressively (a popular video's
metadata is read millions of times). **View counting can't be a naive `UPDATE views=views+1`** on a hot row at 10^6/s —
it's **aggregated** (count in a fast store / stream, batch-flush) and displayed **approximately**. Counting is a
write-amplification problem, covered in [deep dives](deep-dives.md#4-view-counting-at-scale).

---

## Cache hit ratio (the economic lever)

```text
A small % of videos get the vast majority of views (power law).
Cache the hot head at the edge → edge hit ratio can be very high (>90%+) for popular content.
Origin only serves: cold long-tail misses + fills.
```

→ **So what?** **Every 1% of edge hit ratio is enormous $ and origin-load savings** at 30+ Tbps. The design optimizes
for keeping the **hot head resident at the edge** while the **long tail** is served from regional caches / origin. This
is why "CDN strategy" is a first-class deep dive, not an afterthought.

---

## Summary — what the numbers told us

| Number | Value | Design consequence |
|---|---|---|
| **Egress** | **~30+ Tbps** sustained | **CDN-first delivery**; origin is fallback; edge hit ratio = top cost lever |
| Transcode | tens of thousands of video-hours of work/hr | **elastic worker fleet + queue**, segment-parallel, spot compute |
| Storage | multiple PB/day → exabytes | object store + **tiering**; source durable, renditions re-derivable |
| Views | ~10^5–10^6/s | **aggregate/approximate counting**, not hot-row updates |
| Metadata | very high read QPS | **cache** heavily; shard by video |
| Cache hit | >90% for hot head | keep hot head at edge; long tail → regional/origin |

The profile: **a read-heavy, CDN-dominated delivery system fed by a heavy async transcode pipeline.** Design for
**egress served from the edge** and **parallel transcode throughput** — storage and exact counting are not the hard parts.

→ Next: **[API & Data Model](api-data-model.md)**
