# Video Streaming — API & Data Model

> The API splits along the **write/read boundary**: a **resumable upload API** (chunked, into the object store, then async
> processing), and a **playback API** built around a **manifest** (HLS/DASH) that lists renditions + segment URLs — the
> player fetches the manifest, then pulls **immutable segment files from the CDN**. Metadata and view/engagement are
> separate services.

---

## Upload API (resumable, then async)

```text
POST /v1/videos                       # create a video record; returns an upload session
{ "title": "...", "channelId": "c_1" }
→ 201 { "videoId": "v_9", "uploadUrl": "https://upload…/session/abc", "status": "created" }

PUT  {uploadUrl}   (chunked, resumable)   # upload the source in chunks; resume from last byte on failure
  Content-Range: bytes 0-4194303/…
→ 308 Resume Incomplete  (repeat)  →  200 when complete

# On completion the server enqueues the transcode pipeline:
status: created → uploaded → transcoding → ready | failed
```

- **Resumable + chunked** (reuse the [file-storage](../file-storage/api-data-model.md) chunking ideas): large files over
  flaky networks must survive interruption without restarting.
- Upload lands the **source master** directly in the **object store**; completion **enqueues** the transcode job.
- **`status` is async** — the client polls `GET /v1/videos/{id}` or gets a webhook/notification; playback is available
  when `status: ready` (renditions + manifest exist).

```text
GET /v1/videos/{id}      → { videoId, title, channel, status, durationSec,
                             thumbnails:[...], manifestUrl, publishedAt, viewCount(approx) }
```

---

## Playback API (manifest + segments, CDN-served)

The player never streams "a file" — it fetches a **manifest**, then adaptively pulls **segments**.

```text
GET /v1/videos/{id}/manifest.m3u8        # HLS master manifest (or .mpd for DASH) — served/cached at the edge
→ lists available renditions:
   #EXT-X-STREAM-INF: BANDWIDTH=400000,RESOLUTION=426x240   → 240p/rendition_240.m3u8
   #EXT-X-STREAM-INF: BANDWIDTH=1200000,RESOLUTION=1280x720 → 720p/rendition_720.m3u8
   #EXT-X-STREAM-INF: BANDWIDTH=5000000,RESOLUTION=3840x2160 → 4k/rendition_4k.m3u8

GET …/720p/rendition_720.m3u8            # media playlist: ordered list of segment files
   #EXTINF:4.0, seg_000.ts
   #EXTINF:4.0, seg_001.ts   …

GET …/720p/seg_001.ts                    # a ~2–6s SEGMENT file — immutable → CDN-cached forever
```

- The **master manifest** advertises the **renditions**; the player runs **adaptive bitrate** (picks the next segment's
  quality by measured bandwidth/buffer — see [ABR](deep-dives.md#2-adaptive-bitrate-abr)).
- **Segments are small (~2–6 s), immutable, static files** → **infinitely CDN-cacheable**; this is what makes global
  sub-second start + smooth playback possible. The CDN, not your origin, serves ~all of these.
- **HLS vs DASH** are two manifest/segment formats; you typically produce both (or CMAF to share segments) for device
  coverage. See [ABR & formats](deep-dives.md#2-adaptive-bitrate-abr).

---

## Engagement API

```text
POST /v1/videos/{id}/view       # signal a view (deduped/validated server-side; aggregated, approximate)
POST /v1/videos/{id}/like       # like/unlike
GET/POST /v1/videos/{id}/comments?cursor=…    # paginated; write-heavy sub-system
GET  /v1/search?q=…             # title/metadata/transcript index (separate service)
GET  /v1/feed | /v1/recommendations           # recommendations (separate ML system — seam only)
```

---

## Data model

### Access patterns first
1. **Watch page:** fetch a video's metadata + manifest URL + thumbnails → by video id (hot read).
2. **Play:** fetch manifest, then segments → static files from CDN (not the DB).
3. **List a channel's videos** → by channel id, newest first.
4. **Count/read a view count** → high-write increment, approximate read.
5. **Track transcode job state** → by video id (write path).
6. **Search / recommend** → separate indexes/services.

### `videos` (metadata — read-heavy, cache hard, shard by video_id)
```text
video_id (PK) | channel_id (idx) | title | description | duration
status (created|uploaded|transcoding|ready|failed)
source_object_key            # the durable master in the object store
renditions: [ {label:"720p", bitrate, codec, manifest_path} ]   # produced by the pipeline
thumbnails: [urls] | published_at | visibility (public/unlisted/private)
```
- **Read on every watch page** → [cache](../../01-patterns/caching.md) aggressively; **shard by `video_id`**.
- The **rendition list + manifest path is the output of transcode** — written when the video becomes `ready`.

### `transcode_jobs` (write-path state)
```text
job_id (PK) | video_id | segment ranges | target renditions | state per rendition | attempts | worker
```
- Tracks the fan-out of segments × renditions; drives retries/DLQ. Ephemeral-ish; can be pruned after `ready`.

### `view_counts` (aggregated, approximate)
```text
video_id (PK) | approx_count            # backed by a fast counter store + periodic flush (NOT SET count=count+1)
raw view events → stream/queue → aggregation → periodic update of approx_count   (see deep dives)
```

### `channels`, `likes`, `comments`
```text
channels: channel_id (PK) | owner | name | subscriber_count(approx)
likes:    video_id + user_id (PK)        # or aggregated counter
comments: comment_id (PK) | video_id (idx) | user | text | parent_id | created_at    # paginate by video_id
```

### Blobs (object store — the actual bytes)
```text
source master:  object_key → original upload           (durable, 11 nines; re-derive renditions from this)
segments:       …/{video}/{rendition}/seg_NNN.ts       (immutable, CDN-cached; re-derivable → cheaper tier ok)
manifests:      …/{video}/manifest.m3u8 / .mpd         (small, edge-cached)
thumbnails:     …/{video}/thumb_*.jpg
```

---

## SQL or NoSQL? (per store)

- **`videos` metadata → NoSQL/wide-column or sharded relational**, keyed by `video_id`, **heavily cached**. Read-dominated
  point lookups; consistency needs are modest (a newly-`ready` flag propagating in seconds is fine).
- **`transcode_jobs` → KV/relational**, write-path state with retries.
- **`view_counts` → fast counter store** (in-memory/stream-aggregated), flushed to the DB periodically — **never** a
  hot-row `UPDATE`.
- **`comments` → wide-column** partitioned by `video_id` (write-heavy, time-ordered), like a feed.
- **Blobs → object store**, CDN-fronted; segments/manifests immutable.

> **Interview line:** *"Playback isn't 'stream a file' — the player fetches an HLS/DASH manifest that lists renditions,
> then adaptively pulls small immutable segment files straight from the CDN. My DB only holds metadata (keyed by video id,
> cached hard) and the transcode job state; the bytes live in an object store behind the CDN. Uploads are resumable into
> the object store and processed asynchronously — the video flips to `ready` once renditions and the manifest exist. View
> counts are aggregated approximately, never a hot-row increment."*

→ Next: **[Architecture](architecture.md)**
