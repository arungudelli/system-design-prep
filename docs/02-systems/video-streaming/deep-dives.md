# Video Streaming — Deep Dives

The interview lives here: the **transcode pipeline**, **adaptive bitrate**, **CDN delivery strategy**, **view counting at
scale**, **thumbnails**, and the **recommendations** and **live-streaming** seams.

---

## 1. The transcode pipeline (the write-path fan-out)

One upload becomes dozens of outputs. The pipeline is a **segment-parallel fan-out**:

```text
source master → SPLIT into segments (e.g. by GOP / ~few-second chunks)
             → for each segment × each rendition (240p…4K, bitrates, codecs): ENCODE (parallel workers)
             → PACKAGE into HLS/DASH: per-rendition media playlists + master manifest
             → write to object store → mark video READY
```

- **Why segment-parallel:** a 2-hour 4K movie encoded on one CPU takes forever. Splitting into segments lets **thousands
  of workers encode in parallel**, so wall-clock time is ~one segment's encode, not the whole film. Segments are stitched
  via the manifest.
- **The renditions ladder:** a set of (resolution, bitrate) pairs (the "bitrate ladder") covering phone-on-3G to TV-on-
  fiber. Plus **multiple codecs** (H.264 for compatibility, H.265/AV1/VP9 for efficiency) — a matrix of outputs.
- **Elastic + spot compute:** the workload is **bursty, embarrassingly parallel, and retryable** → autoscale a
  [queue + worker](../../01-patterns/queues-workers.md) fleet on backlog, run on **cheap preemptible/spot** instances (an
  interrupted segment just re-queues). Idempotent per (video, segment, rendition) so retries are safe.
- **Prioritization:** short videos / popular creators / re-encodes can get priority lanes so they're not stuck behind a
  giant batch.
- **Ready semantics:** the video is playable when renditions + manifest exist. You can publish progressively (low
  renditions first) or wait for the full ladder.

> **Interview line:** *"Transcoding is a segment-parallel fan-out: split the source into chunks, encode each chunk into
> every rendition on an elastic spot-worker fleet behind a queue, then package into HLS/DASH. Segment parallelism means a
> two-hour movie's wall-clock time is one segment's encode, not the whole film, and spot compute is fine because each
> segment is idempotent and retryable."*

---

## 2. Adaptive bitrate (ABR)

Smooth playback on a changing network is a **client-driven** protocol, not a server feature.

- **The manifest advertises options.** The master manifest (HLS `.m3u8` / DASH `.mpd`) lists each rendition with its
  bandwidth + resolution. The server exposes choices; it doesn't decide.
- **The player picks per segment.** For each ~2–6 s segment, the player estimates **available bandwidth** and its
  **buffer level**, then requests the next segment at the **highest quality it can sustain** — upshifting when the network
  is good, **downshifting to avoid a stall** when it degrades. Stalls (rebuffering) are the worst outcome, so ABR errs
  toward not stalling.
- **Fast start:** begin at a **low bitrate** so the first frame appears in <1–2 s, then ramp up as buffer fills. Start-up
  latency and quality trade off; most players start conservative.
- **Segment size trade:** short segments → faster adaptation + faster start but more request overhead + slightly worse
  compression; long segments → the reverse. ~2–6 s is the usual sweet spot.
- **HLS vs DASH vs CMAF:** two manifest/segment standards (HLS = Apple ecosystem, DASH = elsewhere). **CMAF** lets both
  share one set of segments (produce once, package twice) — a storage/egress win.

> **Interview line:** *"Adaptive bitrate is client-driven: the manifest lists renditions, and for each short segment the
> player picks the best quality its measured bandwidth and buffer can sustain, downshifting before it stalls. It starts
> low for a sub-second first frame, then ramps up. I'd use CMAF so HLS and DASH share one set of segments."*

---

## 3. CDN delivery strategy

At ~30+ Tbps, **the CDN is the system.** Your origin must serve almost nothing.

- **Tiered caching:** **edge** (closest to viewer) → **regional/shield** cache → **origin** (object store). Most requests
  hit the edge; misses pull from regional; only true cold misses reach origin. Each tier shields the next.
- **Segments are perfect cache objects:** small, **immutable**, content-stable URLs → cache **forever**, no invalidation.
  This is what makes edge hit ratios >90% for popular content achievable.
- **Request coalescing (origin protection):** when many viewers miss the same segment at once (a video goes viral), the
  cache **collapses concurrent misses into a single origin fetch** and fans the response out — so a million viewers cause
  ~one origin read, not a million. Critical for surviving virality.
- **Hot head vs long tail:** a power law — a few videos get most views. Keep the **hot head resident at the edge**; the
  **cold long tail** is served from regional caches / origin (and can live on cheaper storage tiers). You don't need every
  video at every edge.
- **Multi-CDN (top stop):** use multiple CDNs with **steering** (route each viewer to the best/cheapest CDN by
  performance, cost, capacity) for resilience and negotiating leverage — see [principal deep dive](principal-deep-dive.md).
- **Pre-warming:** for known spikes (a premiere, a scheduled drop), **push content to edges in advance** so the first
  viewers don't all miss.

> **Interview line:** *"Delivery is CDN-first with tiered edge→regional→origin caching. Segments are immutable, so they
> cache forever with no invalidation, and request coalescing collapses a viral cache-miss storm into a single origin
> fetch. I keep the hot head resident at the edge and let the long tail fall back to regional/origin, and at scale I'd run
> multi-CDN with steering for cost and resilience."*

---

## 4. View counting at scale

Billions of increments/day to hot rows — a naive `UPDATE videos SET views = views + 1` is impossible (lock contention on
a viral video's row melts the DB).

- **Aggregate, don't hot-update.** Emit view **events** to a **stream/queue**; a stream processor (or per-shard counters
  in a fast store like Redis) **aggregates**, and a periodic job **flushes** an **approximate total** to the metadata DB.
  Display is **eventually consistent** — a slightly stale count is fine.
- **Dedup / validate views.** A "view" has rules (min watch time, one per user/session per window, bot filtering) →
  [idempotency](../../01-patterns/idempotency.md) on (user/session, video, window) so refreshes and bots don't inflate it.
- **Approximate structures** where exactness doesn't matter: **HyperLogLog** for unique viewers, sampling/probabilistic
  counters for extreme scale. Exact counts only where money depends on it (monetization uses a separate, auditable path).
- **Sharded counters** to avoid a single hot key: increment one of N shards, sum on read.

> **Interview line:** *"I never hot-update a view row. View events go to a stream, get deduped and aggregated (sharded
> counters or a stream processor), and an approximate total is flushed periodically — display is eventually consistent.
> Unique viewers via HyperLogLog. Exact, auditable counting is a separate path reserved for monetization."*

---

## 5. Thumbnails & preview sprites

- **Thumbnails** are generated in the same pipeline (extract frames at intervals; let creators pick one). Small images →
  object store + CDN + [cache](../../01-patterns/caching.md).
- **Scrubbing previews** (the little images when you hover the timeline) are a **sprite sheet** of thumbnails referenced by
  a small index — generated at transcode time. Cheap to serve, big UX win.

---

## 6. Search & recommendations (the seams)

- **Search:** index title, description, tags, and (powerfully) the **auto-transcribed transcript** into a search service
  (inverted index). Its own system — name it, design the ingestion seam (transcode emits a transcript → indexer), defer
  internals.
- **Recommendations:** a large ML system — **candidate generation** (retrieve plausible videos) + **ranking** (score by
  predicted watch/engagement) fed by watch history, embeddings, and context. Name the two stages and the data seam (watch
  events → feature store → model); **defer the ML**. This is where interviewers let you show you know the shape without
  rat-holing.

---

## 7. Live streaming (a distinct low-latency mode)

Live flips the pipeline from batch to real-time:

- **Ingest** the broadcaster's stream (RTMP/SRT/WebRTC) → **transcode + segment on the fly** → push segments to the CDN
  **as they're produced** → viewers play the growing manifest (a sliding window of recent segments).
- **Latency target** (glass-to-glass) ranges from ~10–30 s (standard HLS) down to **low-latency HLS/DASH** or WebRTC for
  seconds — a real trade of latency vs scale vs cost.
- **DVR / VOD conversion:** a live stream can be recorded and become a normal VOD afterward.
- Same CDN + ABR delivery model; the **hard part is the real-time encode/segment loop and low-latency manifests.** A
  different enough problem to call out as its own system.

---

## 8. DRM & content protection (delivery-layer seam)

- Premium catalogs need **encrypted segments + license/key servers** (Widevine/FairPlay/PlayReady) so only authorized
  players decrypt. Segments are encrypted at packaging; the player fetches a license to play. Note it as a delivery-layer
  concern; UGC platforms often skip it. Incompatible with naive open caching (keys are per-user), but the encrypted
  segments themselves still CDN-cache.

→ Next: **[Trade-offs](tradeoffs.md)**
