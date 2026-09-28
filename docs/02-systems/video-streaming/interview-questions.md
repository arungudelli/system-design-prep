# Video Streaming — Interview Questions

> Answer each **out loud first**, then expand and self-critique against the
> [critique lens](../../00-framework/staff-level-thinking.md#how-your-answer-gets-graded-the-critique-lens).

---

## Core

**Q1. What's the overall shape of this system?**
<details markdown="1"><summary>Model answer</summary>

**Two systems joined by an object store.** A **write path**: resumable chunked upload → async **transcode pipeline**
(segment-parallel fan-out into a bitrate ladder of renditions) → package HLS/DASH → mark `ready`. A **read path**:
**CDN-first** delivery where the player fetches a **manifest** and adaptively pulls immutable **segment files** from edge
caches, plus cached, sharded **metadata** and an **engagement** service with approximate view counts. See
[architecture](architecture.md).
</details>

**Q2. Why transcode into multiple renditions instead of storing one file?**
<details markdown="1"><summary>Model answer</summary>

Viewers span phone-on-3G to TV-on-fiber. One bitrate either stalls the slow network or looks bad on the fast one. Multiple
renditions (a **bitrate ladder** × codecs) let **adaptive bitrate** give each viewer the best quality their network
sustains. The extra transcode + storage is cheap relative to the playback-quality win, and it's a one-time write cost
amortized over millions of reads. See [pipeline](deep-dives.md#1-the-transcode-pipeline-the-write-path-fan-out).
</details>

**Q3. Walk the upload-to-playable path.**
<details markdown="1"><summary>Model answer</summary>

Resumable chunked upload lands the **source master** in the object store → completion **enqueues** a transcode job →
workers **split into segments** and **encode each segment × each rendition in parallel** on an elastic spot fleet →
**package** HLS/DASH (segments + media playlists + master manifest) → write to object store → metadata service records the
rendition list + manifest URL and flips `status: ready`. Async throughout — the uploader waits minutes, not seconds. See
[flow](architecture.md#10-request--data-flow).
</details>

**Q4. How does playback actually work — what does the player fetch?**
<details markdown="1"><summary>Model answer</summary>

Not "a file." The player fetches the **master manifest** (lists renditions), picks a starting rendition (often low for a
fast first frame), fetches that rendition's **media playlist** (an ordered list of ~2–6 s **segment files**), then pulls
segments sequentially from the **CDN edge**. After each segment it measures throughput/buffer and **adapts the next
segment's quality**. Segments are immutable → cached forever at the edge. See [ABR](deep-dives.md#2-adaptive-bitrate-abr).
</details>

**Q5. What's the dominant bottleneck and cost?**
<details markdown="1"><summary>Model answer</summary>

**Egress bandwidth** — tens of Tbps sustained — by a wide margin. That's *why the system is a CDN*: bytes are served from
**edge caches near viewers**, not your origin. Edge **cache hit ratio** and **encoding efficiency (per-title encoding)**
are the top cost levers. Transcode compute is second. Raw storage and exact view counts are not the hard parts. See
[capacity](capacity.md#egress-bandwidth-the-dominant-cost-size-this-first).
</details>

---

## Delivery & scale

**Q6. How do you serve ~30 Tbps of egress and protect your origin?**
<details markdown="1"><summary>Model answer</summary>

**CDN-first with tiered caching** (edge → regional → origin). Immutable segments cache forever with no invalidation, so
hot content sits at the edge (>90% hit ratio). **Request coalescing** collapses a viral cache-miss storm into a single
origin fetch. **Pre-warm** edges for known spikes. Keep the **hot head at the edge**, let the long tail fall to
regional/origin. At scale, **multi-CDN steering** for capacity/resilience/cost. Origin serves almost nothing. See
[CDN](deep-dives.md#3-cdn-delivery-strategy).
</details>

**Q7. How do you scale transcoding for a firehose of uploads?**
<details markdown="1"><summary>Model answer</summary>

An **elastic worker fleet** behind a [queue](../../01-patterns/queues-workers.md), autoscaled on backlog, on **spot/
preemptible** compute — the workload is bursty, embarrassingly parallel, and retryable. **Segment-level parallelism** so a
2-hour movie's wall-clock is one segment's encode, not the whole film. Idempotent per (video, segment, rendition), retries
→ DLQ. Priority lanes for short/popular videos. See [pipeline](deep-dives.md#1-the-transcode-pipeline-the-write-path-fan-out).
</details>

**Q8. How do you count views at billions/day?**
<details markdown="1"><summary>Model answer</summary>

Never a hot-row `UPDATE`. View **events → stream/queue → aggregate** (sharded counters or a stream processor) → **periodic
flush** of an **approximate** total; display is eventually consistent. **Validate + dedup** views (min watch time, one per
user/session/window, bot filtering) via [idempotency](../../01-patterns/idempotency.md). **HyperLogLog** for unique
viewers. Exact, auditable counting is a **separate path** for monetization only. See [view counting](deep-dives.md#4-view-counting-at-scale).
</details>

---

## Reliability & data

**Q9. What must never be lost, and how do you guarantee it?**
<details markdown="1"><summary>Model answer</summary>

The **source master** — everything else (renditions, segments, thumbnails) is **re-derivable** from it. So the master gets
the strongest durability (object store **erasure coding + cross-region replication**, ~11 nines); renditions can live on
cheaper tiers or be regenerated. Uploads are resumable so they're never lost mid-transfer, and transcode failures retry
against the safe master. See [failure scenarios](failure-scenarios.md).
</details>

**Q10. A video goes viral in minutes. What happens?**
<details markdown="1"><summary>Model answer</summary>

The first viewers miss cold segments at the edge → **request coalescing** collapses those concurrent misses into one
origin fetch per segment; once cached, the **edge serves the millions** that follow. Immutable segments make this safe.
Tiered caching shields origin; if it's a *known* spike (premiere), **pre-warm** edges first. Metadata is cached and view
counting is off-path, so neither becomes the bottleneck. See [origin overload](failure-scenarios.md#origin-overload-from-a-viral-video-cache-miss-storm).
</details>

**Q11. A viewer's network gets worse mid-video. What happens?**
<details markdown="1"><summary>Model answer</summary>

**Adaptive bitrate downshifts** to a lower rendition to keep playing — **degrade quality, never stall**. The player
watches its buffer + measured bandwidth and picks the next segment's quality accordingly; it errs toward not rebuffering
because stalls are the worst quality outcome. See [ABR](deep-dives.md#2-adaptive-bitrate-abr).
</details>

---

## Staff-level curveballs

**Q12. How would you cut the egress bill (the dominant cost)?**
<details markdown="1"><summary>Model answer</summary>

Three levers: **(1) maximize edge cache hit ratio** (keep the hot head resident; tiered caching; every 1% is huge at 30
Tbps); **(2) per-title / per-scene encoding** — analyze each video and choose the optimal bitrate ladder so you send fewer
bits for the same quality; **(3) multi-CDN steering** to route each viewer to the cheapest CDN that meets quality. Plus
efficient codecs (AV1/H.265) where devices support them. See [trade-offs](tradeoffs.md#8-encoding-fixed-ladder-vs-per-titleper-scene-encoding).
</details>

**Q13. How does live streaming differ from VOD here?**
<details markdown="1"><summary>Model answer</summary>

Live flips the pipeline from batch to **real-time**: ingest (RTMP/SRT/WebRTC) → **transcode + segment on the fly** → push
segments to the CDN as produced → viewers play a **sliding-window manifest**. Same CDN + ABR delivery, but the hard part is
the **low-latency encode/segment loop** and low-latency manifests (LL-HLS/DASH or WebRTC for seconds of glass-to-glass).
It's a distinct mode; a recording can convert to VOD afterward. See [live](deep-dives.md#7-live-streaming-a-distinct-low-latency-mode).
</details>

**Q14. Where does this reuse patterns and systems you've studied?**
<details markdown="1"><summary>Model answer</summary>

**Chunked upload + object store:** from [file storage](../file-storage/README.md). **Queues + workers:** the transcode
fan-out. **Caching/CDN:** the entire read path. **Sharding:** metadata by video. **Idempotency:** transcode jobs + view
dedup. **Rate limiting:** upload quotas. **Notification/feed ideas:** comments + recommendations seams. It's the
read-heavy, CDN-first cousin of file storage — same object store, opposite emphasis (delivery + processing vs sync).
</details>

**Q15. What changes at 10× and 100×?**
<details markdown="1"><summary>Model answer</summary>

**10×:** more transcode capacity, bigger multi-CDN footprint, harder hot-content caching. **100×:** **multi-CDN with
active steering**, **per-title encoding** everywhere (egress $), aggressive **storage tiering**, **edge compute**
(personalization/ads at the edge), and mature **live** infrastructure. The CDN-first + async-pipeline backbone holds;
pressure is on egress cost/resilience and encoding efficiency. See [principal deep dive](principal-deep-dive.md).
</details>

**Q16. When would you redesign this system?**
<details markdown="1"><summary>Model answer</summary>

When assumptions break: **live/real-time becomes primary** (a low-latency ingest+delivery system, not VOD batch);
**interactivity** (real-time watch-together, cloud gaming) needs WebRTC-grade latency; the product becomes
**recommendation-first** (the feed/ML system dominates, video is a backend); or **strict DRM/compliance** reshapes the
delivery + key-management layer. Absent those, the CDN-first VOD design holds.
</details>

---

## Self-scoring

Grade against the [15-point critique lens](../../00-framework/staff-level-thinking.md#how-your-answer-gets-graded-the-critique-lens):
did you separate the async write path from the CDN read path, transcode into a bitrate ladder via a segment-parallel
elastic fleet, deliver CDN-first with tiered caching + request coalescing, make playback client-driven ABR, count views
approximately off-path, protect the durable source (renditions re-derivable), and name the egress bill as the dominant
cost with per-title encoding + multi-CDN as levers? Note your weakest area and drill it.

← Back to **[README](README.md)** · Next: **[Cheatsheet](cheatsheet.md)**
