# Video Streaming — Principal-Level Deep Dive (The Stops)

> The **"never be blind"** companion to the base Video Streaming docs. The system as a **staircase of stops** — smallest
> defensible design first, each bigger stop with its **exact trigger** — ending in the **top-stop (max-scale /
> deliberately over-engineered)** architecture plus a **reconciliation table** for when each heavy component is overkill
> and should be **removed**.
>
> Read the base first: [Requirements](requirements.md) · [Architecture](architecture.md)
> · [Deep Dives](deep-dives.md) · [Trade-offs](tradeoffs.md).
>
> **Say this in an interview:** *"I'll start with a single-rendition upload-and-serve and grow it one bottleneck at a
> time. I know the multi-CDN-steered, per-title-encoded, live-capable, edge-compute version — but each piece only earns
> its place when a specific ceiling forces it."*

---

## The stops (climb one lever at a time)

| Stop | Scale target | Shape | **Trigger that forces the NEXT stop** |
|---|---|---|---|
| **1 · Upload & serve** | tiny | upload file → transcode to one format → store → stream from a web server | doesn't fit varied devices/networks; server can't serve egress; one CPU/video |
| **2 · Multi-rendition + async pipeline + CDN** | moderate | resumable upload → **queue + worker** transcode into a rendition ladder → HLS/DASH → **CDN** delivery with ABR | global viewers; viral spikes; view write volume; egress cost |
| **3 · CDN-first at scale: tiered caching, coalescing, elastic spot transcode, approximate views (base target)** | 100Ms of viewers | edge→regional→origin tiers, **request coalescing**, autoscaled **spot** transcode, aggregated **approximate** view counts, cached sharded metadata, storage tiering | egress cost + resilience; encoding efficiency; live demand; personalization |
| **4 · Multi-CDN steering + per-title encoding + live + edge compute (top stop)** | global, billions of watch-hrs | **multi-CDN with steering**, **per-title/per-scene encoding**, low-latency **live** pipeline, **edge compute** (personalization/ads/DRM), cross-region everything | (ceiling; real-time interactive video becomes a different system) |

> Most interviews want **Stop 2–3**. Reach into Stop 4 only when the prompt says "global / minimize egress cost /
> live / personalized at the edge." Name the trigger each time.

---

## Top-stop architecture (max-scale)

```text
WRITE
  uploader ─resumable→ [Upload Svc] → [Object Store: SOURCE master, cross-region, 11 nines]
                            │ enqueue
                      [Transcode Queue] → [Elastic SPOT Worker Fleet]
                            - split into segments
                            - PER-TITLE/PER-SCENE analysis → optimal bitrate ladder
                            - encode segment×rendition×codec (H.264/H.265/AV1) in parallel
                            - package CMAF (shared HLS+DASH segments) → [Object Store: renditions, cheaper tier]
                            → [Metadata: rendition list, manifest URLs, status=ready]

READ
  player → [Metadata + cache]  (manifest URL, per-viewer)
  player → [CDN STEERING] → routes to best/cheapest healthy CDN by perf/cost/capacity
                │
        [CDN A edge] [CDN B edge] [CDN C edge] ─miss→ [regional/shield] ─miss→ [Object Store origin]
                │  request coalescing at each tier; immutable segments cache forever
        [EDGE COMPUTE]: personalization, ad insertion, DRM license checks, LL-HLS for live

LIVE (parallel pipeline)
  broadcaster ─RTMP/SRT/WebRTC→ [Live Ingest] → real-time transcode+segment → LL-HLS/DASH → CDN (sliding window)

VIEWS/ENGAGEMENT
  view events → stream processor → sharded/approx counters (HLL for uniques) → periodic flush
  monetization → SEPARATE exact, auditable counting path
```

---

## Deep dives that only matter at the top stop

### Multi-CDN steering (Stop 3 → 4 trigger: **egress cost + single-CDN risk + regional performance**)
- **Why:** one CDN is an outage SPOF, a capacity ceiling, and gives no cost/performance leverage; egress is *the* bill.
- **Design:** integrate multiple CDNs; a **steering** layer routes each viewer (or session) to the **best/cheapest healthy**
  CDN by real-time performance, cost, and capacity signals. Segments are static → any CDN can serve them.
- **Over-engineering watch:** *only when egress $ and resilience justify it.* A single good CDN serves enormous scale;
  **remove** multi-CDN below serious cost/resilience pressure (it adds steering + reconciliation complexity).

### Per-title / per-scene encoding (trigger: **egress cost at scale**)
- **Why:** a fixed bitrate ladder wastes bits on simple content and starves complex content; at 30+ Tbps, bits = money.
- **Design:** analyze each video (or scene) and pick the **optimal ladder** → same perceptual quality, **fewer bits**,
  directly cutting egress. Costs extra analysis compute per upload (worth it — amortized over millions of views).
- **Over-engineering watch:** *only at real egress scale.* A fixed ladder is simpler and fine early; **remove** per-title
  analysis when your egress bill is small.

### Live streaming pipeline (trigger: **real-time broadcast product**)
- **Why:** live is a fundamentally different latency regime than VOD batch.
- **Design:** low-latency **ingest** (RTMP/SRT/WebRTC) → **real-time transcode + segment** → push to CDN as produced →
  **sliding-window LL-HLS/DASH** manifests (or WebRTC for seconds of glass-to-glass). Record → convert to VOD after.
- **Over-engineering watch:** *only if the product is live.* Pure VOD doesn't need it; **remove** the whole live path
  otherwise. See [live](deep-dives.md#7-live-streaming-a-distinct-low-latency-mode).

### Edge compute (trigger: **personalization / ads / DRM at the edge, at scale**)
- **Why:** doing per-viewer work (personalized manifests, ad insertion, DRM license checks, LL-live) at origin adds latency
  and origin load; push it to the edge.
- **Design:** run lightweight logic at CDN edge nodes — manifest personalization, server-guided ABR, ad stitching, token/
  DRM checks — close to the viewer.
- **Over-engineering watch:** *only when per-viewer edge logic matters at scale.* Static delivery needs none; **remove**
  edge compute for a simple VOD catalog.

### Storage tiering & rendition regeneration (trigger: **exabyte long tail + cost**)
- **Why:** most videos + most renditions are rarely watched; keeping everything hot is wasteful.
- **Design:** **tier by access** (hot head hot; cold tail on cheap cold storage); optionally **regenerate rarely-watched
  renditions on demand** from the durable source instead of storing them. Source masters stay durable regardless.
- **Over-engineering watch:** *only at exabyte scale.* Small catalogs keep everything hot; **remove** tiering below that.

---

## Reconciliation table — what forces each heavy component (and when to remove it)

| Component | Base docs (Stop 2–3) | Top stop (Stop 4) | Trigger to ADOPT | When it's OVERKILL → remove |
|---|---|---|---|---|
| CDN | single CDN, tiered caching | **multi-CDN + steering** | egress $ + resilience + regional perf | one good CDN suffices |
| Encoding | fixed bitrate ladder | **per-title / per-scene** | egress cost at scale | small egress bill |
| Live | — (VOD only) | real-time ingest + LL delivery | live/broadcast product | pure VOD |
| Edge logic | static delivery | **edge compute** (personalization/ads/DRM) | per-viewer edge work at scale | static catalog |
| Storage | object store + basic tiering | aggressive tiering + on-demand regen | exabyte long tail + cost | small catalog, all hot |
| Transcode | elastic queue-backed fleet | + priority lanes, multi-codec (AV1) | throughput + device coverage + efficiency | modest upload volume |
| View counting | aggregate + approximate | + HLL uniques, separate monetization path | billions/day + monetization | modest view volume |

> Lead with Stop 2–3 (multi-rendition async pipeline + ABR + CDN-first tiered caching + request coalescing + approximate
> off-path view counting). When pushed: *"for global scale with a serious egress bill, live, and edge personalization,
> here's Stop 4 — multi-CDN steering, per-title encoding, a live pipeline, and edge compute — each mapped to a ceiling, and
> below that ceiling I'd remove it."*

---

## The 60-second Principal summary

> *"It's two systems joined by an object store. The write path is an async, segment-parallel transcode pipeline: resumable
> upload to a durable source master, then an elastic spot-worker fleet splits the video into segments and encodes each into
> a bitrate ladder of renditions, packaged as HLS/DASH — the video is ready when renditions and a manifest exist, and
> because renditions are re-derivable from the master they get cheaper durability. The read path is CDN-first: the player
> fetches a manifest and adaptively pulls small immutable segment files from edge caches, choosing quality per segment by
> measured bandwidth so it starts fast and never stalls. Immutable segments cache forever, request coalescing collapses
> viral miss-storms into a single origin fetch, and I keep the hot head at the edge with the long tail on cheaper tiers.
> Metadata is cached and sharded by video; view counts are aggregated approximately off the playback path. Egress is the
> dominant cost, so pushed to global scale I climb one lever at a time: multi-CDN steering for cost and resilience,
> per-title encoding to send fewer bits, a real-time live pipeline if the product needs it, and edge compute for
> per-viewer logic. Each piece has a named trigger, and below it I'd remove it."*

← Back to **[Video Streaming overview](README.md)** · Base **[deep dives](deep-dives.md)** · **[cheatsheet](cheatsheet.md)**
