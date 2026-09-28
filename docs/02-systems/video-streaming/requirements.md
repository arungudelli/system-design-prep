# Video Streaming — Requirements

## 1. Problem statement (the interview prompt)

> "Design a video platform like **YouTube / Netflix**. Users upload videos; the system processes them so they play
> smoothly on any device and network; hundreds of millions of viewers watch, with fast start-up and quality that adapts
> to their bandwidth. Include metadata, view counts, and basic search — at billions of watch-hours a day and exabytes of
> storage."

Deliberately broad. The three answers that reshape everything: **user-generated upload (YouTube) vs curated catalog
(Netflix)** (drives whether transcode is a firehose or a batch job), **VOD vs live** (live is a much harder latency
problem), and **global scale / device diversity** (drives ABR + multi-CDN). Clarify before drawing boxes.

---

## 2. Clarifying questions

### Product / scope
- **User uploads (YouTube) or a curated catalog (Netflix)?**
  *Why:* UGC means a **continuous, unpredictable transcode firehose** + moderation; a catalog means **offline batch
  encoding** of a known set (you can even hand-tune per-title encoding). Very different write paths.
- **VOD only, or live streaming too?**
  *Why:* **live** flips the pipeline to low-latency real-time (segment-as-you-ingest, seconds of glass-to-glass) — a
  different, harder system. Confirm early; usually start VOD, note live as a seam.
- **Which devices / networks?** phone on 3G to smart-TV on fiber?
  *Why:* device + network diversity is *the* justification for **multiple renditions + adaptive bitrate**.
- **Max video length / size?** minutes, or multi-hour 4K?
  *Why:* drives resumable chunked upload, transcode cost, and storage.
- **Features:** search, recommendations, comments, likes, channels/subscriptions, monetization?
  *Why:* each is a sub-system; recommendations especially is its own world — name it, scope it, defer detail.

### Scale
- **Uploads/day (hours of video)? Views/day and peak concurrent streams? Avg watch time?**
  *Why:* upload hours size the **transcode fleet + storage**; concurrent streams size the **egress/CDN** (the dominant cost).

### Consistency / latency
- **Playback start-up target?** (typically **< ~1–2 s** to first frame.)
  *Why:* justifies CDN-first delivery + fast manifest fetch.
- **Upload-to-available latency?** (minutes is fine — transcode is async.)
  *Why:* the write path is explicitly allowed to be slow; don't block the uploader.
- **View-count accuracy?** exact, or approximate + eventually consistent?
  *Why:* exact counting at billions/day is expensive; approximate is standard for display.

### Reliability
- **Durability of source + renditions?** (never lose an upload; renditions are re-derivable.)
  *Why:* the **source master** must be durable (11 nines); renditions can be **regenerated**, so they're cheaper to lose.

---

## 3. Functional requirements

### Must have
1. **Upload video** — large files, **resumable**, any common format.
2. **Transcode** — process each upload into **multiple renditions** (resolutions/bitrates/codecs) for device+network fit.
3. **Stream/playback** — smooth, fast-start, with **adaptive bitrate** across devices worldwide.
4. **Video metadata** — title, description, owner/channel, thumbnails, rendition list; fetch for a watch page.
5. **View counts & basic engagement** — count views (approximate at scale); likes.
6. **Durable storage** — never lose an uploaded source; renditions re-derivable.

### Nice to have (name, then defer)
- **Search** — index title/metadata/transcript; its own service. Design the seam.
- **Recommendations / home feed** — a large ML system; name it, defer.
- **Comments** — a write-heavy sub-system (reuse feed/queue ideas).
- **Live streaming** — real-time low-latency pipeline; call out as a distinct mode.
- **Monetization / ads** — ad insertion, another pipeline; defer.
- **DRM / content protection** — for premium catalogs; note as a delivery-layer concern.

> **Interview line:** *"Must-haves: resumable upload, an async transcode pipeline producing multiple renditions,
> CDN-delivered adaptive-bitrate playback with sub-second start, video metadata, and approximate view counts, on durable
> storage. I'll design the seams for search, recommendations, comments, and live, and defer their internals. I'd confirm
> UGC-vs-catalog and VOD-vs-live up front, since both reshape the pipeline."*

---

## 4. Non-functional requirements (quantified)

| NFR | Target | Why |
|---|---|---|
| **Playback start-up** | **p95 < ~1–2 s** to first frame | Must feel instant → CDN-first + fast manifest + low-bitrate first segment. |
| **Rebuffer ratio** | **very low** (< ~0.5% of watch time) | Stalls are the #1 quality complaint → adaptive bitrate + edge caching. |
| **Availability (playback)** | **very high**; degrade quality before failing | A viewer should get *some* quality, not an error. |
| **Durability (source)** | **~11 nines** for the master; renditions re-derivable | Losing an upload is unacceptable; renditions can be regenerated. |
| **Upload-to-available** | **minutes** (async) | Transcode is heavy; the uploader mustn't wait. |
| **Scale** | 100Ms of viewers, **exabytes**, billions of watch-hours/day | Sizes CDN egress (dominant), transcode fleet, storage. |
| **View-count accuracy** | **approximate, eventually consistent** for display | Exact counting at this write volume isn't worth the cost. |
| **Global reach** | edge presence worldwide | Latency = distance; serve from near the viewer. |

**Dominant NFRs:** **egress bandwidth / CDN delivery** (the cost + latency center of gravity), **smooth adaptive
playback**, **transcode throughput**, and **source durability**. Raw storage and exact view accuracy are explicitly not
the hard parts.

---

## 5. What we are explicitly NOT building (this pass)

- **The object store & CDN internals** — reuse S3-class storage + a commercial/edge CDN; we design the pipeline and
  delivery strategy *around* them, not the disk/edge layer.
- **The video codecs themselves** — use H.264/H.265/AV1/VP9 encoders; we orchestrate them, we don't implement them.
- **The recommendation ML system** — a whole separate domain; name the seam (candidate generation + ranking), defer.
- **Full live-streaming stack** — note it as a distinct low-latency mode and its pipeline differences; don't build it here.
- **DRM crypto & ad-insertion internals** — mention as delivery-layer concerns; defer.

→ Next: **[Capacity](capacity.md)**
