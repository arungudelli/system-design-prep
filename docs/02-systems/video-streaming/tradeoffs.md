# Video Streaming — Trade-offs

> Dominant requirements: **CDN-served egress at the lowest cost/latency**, **smooth adaptive playback across devices**,
> **transcode throughput**, and **source durability** — with exact view counts and raw storage explicitly *not* the hard
> parts. Judge every trade against those.

---

## 1. Transcode: on-upload (async) vs on-the-fly (per request)

- **On-upload (pre-transcode all renditions):** playback is instant (segments ready); costs storage for renditions
  watched-or-not.
- **On-the-fly (transcode per playback request):** saves storage; but re-encoding on every view is absurd at read-heavy
  scale (you'd transcode the same segment millions of times) and adds latency.

**Choose:** **pre-transcode on upload.** Reads vastly outnumber writes — do the work once, serve millions of times.
On-the-fly only for rarely-watched long-tail renditions as a cost optimization. See [pipeline](deep-dives.md#1-the-transcode-pipeline-the-write-path-fan-out).

---

## 2. Delivery: adaptive bitrate vs single "best" stream

- **Single stream:** simple; but one bitrate can't serve both a phone on 3G (stalls) and a TV on fiber (looks bad).
- **ABR (multiple renditions):** each viewer gets the best quality their network sustains; costs extra transcode + storage.

**Choose:** **ABR.** Device + network diversity is the whole point; the extra renditions are cheap relative to the
playback-quality win. See [ABR](deep-dives.md#2-adaptive-bitrate-abr).

---

## 3. Delivery infra: CDN vs serve-from-origin

- **Origin serving:** simple; but ~30+ Tbps cannot come from your servers, and distant viewers get high latency.
- **CDN-first:** serve from edge caches near viewers; origin only handles misses.

**Choose:** **CDN-first, always.** If your origin serves video bytes at scale, the design is wrong. Origin is a fallback.
See [CDN](deep-dives.md#3-cdn-delivery-strategy).

---

## 4. Single CDN vs multi-CDN

- **Single:** simpler integration/billing; but a CDN outage = total delivery outage, and no cost/perf leverage.
- **Multi-CDN + steering:** resilience, cost optimization, and per-region performance; adds steering logic + reconciliation.

**Choose:** **single CDN early, multi-CDN at scale.** The egress bill + resilience justify multi-CDN once you're large;
steering routes each viewer to the best/cheapest option. A top-stop concern.

---

## 5. View counting: exact vs approximate

- **Exact:** every view counted precisely; requires hot-row updates or heavy coordination — melts at 10^6/s.
- **Approximate (aggregate + eventual):** stream + aggregate + periodic flush; a slightly stale display count.

**Choose:** **approximate for display**, with a **separate exact/auditable path only for monetization** (where money
depends on it). See [view counting](deep-dives.md#4-view-counting-at-scale).

---

## 6. Segment length: short vs long

- **Short (~2 s):** faster start + faster ABR adaptation; more requests + slightly worse compression.
- **Long (~10 s):** better compression + fewer requests; slower start + coarser adaptation.

**Choose:** **~2–6 s** as the balance; shorter if start-up latency / live is the priority, longer for VOD efficiency.

---

## 7. Storage tiering: keep all renditions hot vs tier / regenerate cold

- **All hot:** uniform fast access; expensive for the huge rarely-watched long tail + every rendition.
- **Tier / regenerate:** cold videos + rarely-used renditions on cheaper storage, or **regenerate on demand** from the
  durable source.

**Choose:** **tier by access**; keep source masters durable, keep popular renditions hot, and archive/regenerate the cold
long tail. Renditions are re-derivable, so losing a cold one is cheap. See [capacity](capacity.md#storage-source--renditions).

---

## 8. Encoding: fixed ladder vs per-title/per-scene encoding

- **Fixed bitrate ladder:** simple, one-size-fits-all; wastes bits on simple content, starves complex content.
- **Per-title / per-scene encoding:** analyze each video (or scene) and choose the optimal bitrate ladder → **same quality,
  fewer bits** → direct **egress cost** win; costs extra analysis compute.

**Choose:** **fixed ladder early, per-title at scale.** At 30+ Tbps, shaving bits per stream is enormous $ — worth the
analysis cost. A top-stop lever.

---

## 9. Publish: wait for full ladder vs progressive availability

- **Wait for full ladder:** consistent experience; longer upload-to-available.
- **Progressive (publish low renditions first):** available sooner; early viewers get fewer quality options.

**Choose:** **product-driven** — progressive for creator immediacy (YouTube), full-ladder for premium catalogs (Netflix).

---

## 10. Upload: resumable-chunked vs single-shot

- **Single-shot:** simple; a dropped connection restarts a multi-GB upload.
- **Resumable chunked:** survives flaky networks; resume from last byte.

**Choose:** **resumable chunked** for anything but tiny files — reuse the [file-storage](../file-storage/deep-dives.md#1-chunking-the-decision-that-unlocks-everything)
chunking approach.

---

## The one-paragraph summary (say this)

> *"Write and read are two different systems joined by an object store. Upload is resumable and chunked into the object
> store, then an async, segment-parallel transcode pipeline on an elastic spot-worker fleet encodes each upload into a
> bitrate ladder of renditions and packages HLS/DASH — the video is 'ready' when renditions plus a manifest exist. Reads
> are CDN-first: the player fetches a manifest and adaptively pulls small immutable segment files from edge caches, picking
> quality per segment by measured bandwidth so it starts fast and never stalls. Immutable segments cache forever, request
> coalescing collapses viral miss-storms into one origin fetch, and I keep the hot head at the edge while the long tail
> falls back to regional/origin on cheaper storage. Metadata is cached and sharded by video; view counts are aggregated
> approximately, never hot-row updates. At scale I add multi-CDN steering and per-title encoding to cut the egress bill."*

→ Next: **[Failure Scenarios](failure-scenarios.md)**
