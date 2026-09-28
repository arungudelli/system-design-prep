# Video Streaming (YouTube / Netflix)

> Design a video platform — users **upload** videos of any size, the system **transcodes** each into many
> resolutions/bitrates, and **hundreds of millions of viewers stream** them smoothly on any device and network, starting
> in **under a second**, adapting quality to bandwidth in real time — plus metadata, view counts, search, and
> recommendations — at a scale of **billions of watch-hours a day** and **exabytes** of stored video.

The naive version ("store the uploaded file, stream it back") misses the entire interview. The hard parts are **not**
storing bytes — an object store does that. They are: an **asynchronous transcode pipeline** that fans one upload into
dozens of renditions, **adaptive-bitrate delivery** so playback survives a shaky network, a **CDN-centric architecture**
where the CDN — not your origin — serves ~all the bytes, and **counting views** at a scale where the count itself is a
distributed-systems problem.

---

## Why it's a great system to study

- **It's the canonical read-heavy, CDN-first system** — egress dwarfs everything; the entire design bends around
  **serving bytes from the edge**, not from your servers. If your origin serves video, you've already lost.
- **The transcode pipeline is a beautiful fan-out** — one upload → split into segments → transcode each into N
  renditions in parallel → package for HLS/DASH. It reuses [queues + workers](../../01-patterns/queues-workers.md) at scale.
- **Adaptive bitrate is where playback becomes a protocol** — the client, not the server, decides which quality to fetch
  next based on measured bandwidth; the server just exposes a **manifest** of options.
- **View counting is deceptively hard** — billions of increments to hot rows is a write-amplification + accuracy trade,
  a great "how would you actually count this" discussion.
- **It composes prior systems** — [chunking + object store](../file-storage/README.md) for upload/storage,
  [queues](../../01-patterns/queues-workers.md) for the pipeline, [caching](../../01-patterns/caching.md)/CDN for delivery,
  and a metadata DB [sharded](../../01-patterns/sharding.md) by video.

---

## Read in this order

1. **[Requirements](requirements.md)** — problem, clarifying questions, functional + NFRs
2. **[Capacity](capacity.md)** — storage, transcode compute, the dominant egress/CDN bandwidth, views QPS
3. **[API & Data Model](api-data-model.md)** — resumable upload, playback manifest (HLS/DASH), metadata & view schema
4. **[Architecture](architecture.md)** — upload → object store → transcode pipeline → packaging → CDN → player
5. **[Deep Dives](deep-dives.md)** — transcode pipeline, adaptive bitrate, CDN strategy, view counting, thumbnails, recs, live
6. **[Trade-offs](tradeoffs.md)** — the decisions, both sides
7. **[Failure Scenarios](failure-scenarios.md)** — transcode failure, CDN miss/origin overload, hot video, view accuracy
8. **[Interview Questions](interview-questions.md)** — attempt first, then reveal
9. **[Cheatsheet](cheatsheet.md)** — 1-page revision
10. **[★ Principal Deep Dive](principal-deep-dive.md)** — the **staged stops** (single rendition → global) + max-scale variant
    (per-title encoding, multi-CDN steering, live streaming, edge compute) with a reconciliation table

---

## The 30-second version (know this cold)

- **Write path and read path are totally different systems.** **Upload + transcode** is a heavy **async batch pipeline**
  (minutes of work per video); **playback** is a **read-heavy, latency-sensitive CDN** problem. Design them separately.
- **Upload is resumable + chunked.** Big files over flaky networks → chunked, resumable upload straight into an
  **object store** (reuse the [file-storage](../file-storage/README.md) chunking ideas). Then hand off to the pipeline.
- **Transcode fans out.** Split the source into segments; transcode each into **multiple renditions** (e.g.
  240p→4K, several bitrates, multiple codecs) **in parallel** via a [queue + worker](../../01-patterns/queues-workers.md)
  fleet; **package** into HLS/DASH segments + a **manifest**. The video is "ready" when renditions + manifest exist.
- **Delivery is CDN-first.** Segments are static, immutable files served from **CDN edge caches** near the viewer;
  your **origin is a fallback**, not the hot path. This is what makes global sub-second start possible.
- **Adaptive bitrate (ABR).** The player fetches a **manifest** listing renditions, then **picks the next segment's
  quality based on measured bandwidth/buffer** — upshifting on fast networks, downshifting to avoid stalls. The
  intelligence is in the client.
- **Metadata & views are their own services.** Video metadata (title, owner, rendition list) in a
  [sharded](../../01-patterns/sharding.md) DB + cache; **view counts** are aggregated approximately at massive write
  volume, not a naive `UPDATE … SET views=views+1`.
- **Bottlenecks:** **egress bandwidth (CDN)** by a mile, then **transcode compute**, then **view-count write
  amplification** and **hot videos** (a viral upload). Raw storage is commodity.
