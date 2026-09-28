# Video Streaming — Failure Scenarios

> Themes: **never lose the source master** (renditions are re-derivable, the upload isn't), **protect the origin** (the CDN
> absorbs load; a miss-storm must never melt origin), **degrade quality before failing playback**, and **the async write
> path can fail slowly and retry** without hurting viewers.

---

## Upload interrupted mid-transfer

- **Impact:** a large upload drops partway.
- **Contain:** **resumable chunked upload** — the client resumes from the last acknowledged byte, not from zero. Nothing is
  processed until the source is complete, so a partial upload is just staged bytes. See [upload](api-data-model.md#upload-api-resumable-then-async).

## A transcode job (or one rendition) fails

- **Impact:** some renditions don't get produced.
- **Contain:** the job is **idempotent per (video, segment, rendition)** → **retry with backoff**; after N → **DLQ +
  alert**. The **source master is durable**, so any rendition can be **re-derived** later. The video can go `ready` with the
  renditions that succeeded (progressive) or stay `processing`. A failed encode never risks the original. See
  [pipeline](deep-dives.md#1-the-transcode-pipeline-the-write-path-fan-out).

## Transcode backlog explodes (upload spike)

- **Impact:** upload-to-available latency grows.
- **Contain:** the **queue absorbs the burst**; the **elastic worker fleet autoscales** on backlog (spot compute). Priority
  lanes keep short/popular videos fast. Uploaders already expect minutes, so this degrades gracefully — slower, not broken.

## Origin overload from a viral video (cache-miss storm)

- **Impact:** a video suddenly goes viral; millions miss the same cold segments at once → origin hammered.
- **Contain:** **request coalescing** collapses concurrent misses for the same segment into **one origin fetch**; **tiered
  caching** (edge → regional → origin) shields origin; once cached, the edge serves everyone. **Pre-warm** edges for known
  spikes (premieres). The immutable-segment model means one fetch satisfies millions. See [CDN](deep-dives.md#3-cdn-delivery-strategy).

## CDN edge/node failure or a whole CDN outage

- **Impact:** viewers on that edge/CDN can't fetch segments.
- **Contain:** other edges/regional caches serve; **multi-CDN steering** (at scale) routes viewers to a healthy CDN.
  Single-CDN early is a risk to name. Because segments are static + cacheable, failover is just re-routing fetches.

## Viewer's network degrades mid-playback

- **Impact:** bandwidth drops; risk of stalling.
- **Contain:** **adaptive bitrate downshifts** to a lower rendition to keep playing — **degrade quality, don't stall**. The
  buffer + ABR logic is designed exactly for this; a stall (rebuffer) is the worst outcome and ABR errs against it. See
  [ABR](deep-dives.md#2-adaptive-bitrate-abr).

## Metadata service slow/down

- **Impact:** watch pages can't load video info / manifest URL.
- **Contain:** metadata is **cached hard** (popular videos read from cache); read replicas + sharding by video isolate blast
  radius. Degrade to cached/stale metadata rather than failing the watch page. The manifest + segments themselves are on the
  CDN, independent of the metadata DB.

## View-count store overwhelmed or lags

- **Impact:** counts stop updating or spike.
- **Contain:** counting is **off the playback path** — a lagging/failed counter **never affects playback**. Counts are
  **approximate + eventually consistent** anyway; buffer view events in the stream and catch up. Worst case: a temporarily
  stale number. See [view counting](deep-dives.md#4-view-counting-at-scale).

## Bot / fraudulent views

- **Impact:** inflated counts (ad fraud, gaming rankings).
- **Contain:** **validate views** (min watch time, one per user/session/window, bot detection) and **dedup**
  ([idempotency](../../01-patterns/idempotency.md)); the **auditable monetization path** is stricter than the display count.

## Storage bit rot / lost rendition

- **Impact:** a stored segment/rendition is corrupted or lost.
- **Contain:** object store **erasure coding + replication** (11 nines) repairs bit rot; a lost **rendition** is
  **re-derivable from the durable source**. Only the **source master** is irreplaceable, so it gets the strongest
  durability. Corruption is detectable (checksums) and repaired.

## Region failure

- **Impact:** lose transcode capacity / an origin region.
- **Contain:** **cross-region replicated** object store + multi-region transcode; CDN is global and keeps serving cached
  content; uploads/transcode fail over to another region. Committed sources are durable across regions.

## "Poison" upload (corrupt/unsupported file, or abuse)

- **Impact:** a file that fails every transcode attempt, or malicious content.
- **Contain:** cap retries → **DLQ + surface to the uploader** ("we couldn't process this"); **content moderation** (scan/
  ML/human review) gates publishing for UGC; never loop forever on a bad file.

---

## Failure-handling toolkit used here

`resumable upload` (survive interruption) · `durable source + re-derivable renditions` (never lose the master) ·
`idempotent transcode + retry → DLQ` (write-path resilience) · `elastic queue-backed fleet` (backlog spikes) ·
`request coalescing + tiered caching + pre-warm` (origin protection) · `multi-CDN steering` (CDN outage) · `ABR downshift`
(degrade quality not playback) · `cached sharded metadata` · `approximate off-path view counting` · `view validation +
dedup` (fraud) · `erasure coding + cross-region` (durability).

## Priorities (say this)

> *"My non-negotiables: never lose the source master, and never fail playback outright. Everything else degrades — a failed
> transcode retries and re-derives from the durable source; a viral miss-storm is collapsed by request coalescing so origin
> sees one fetch; a bad network makes ABR downshift quality rather than stall; a lagging view counter never touches
> playback because counting is off-path and approximate anyway. Renditions are re-derivable so they get cheaper durability
> than the irreplaceable master, and the CDN keeps serving cached content through origin and region failures."*

→ Next: **[Interview Questions](interview-questions.md)**
