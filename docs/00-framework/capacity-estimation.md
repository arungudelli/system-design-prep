# Capacity Estimation

> Back-of-the-envelope math has one job: **turn vague requirements into a number that changes a design
> decision.** If a calculation doesn't change what you build, don't do it. Interviewers want to see that
> you know *which* numbers matter — not that you can multiply.

---

## The mindset

- **Order of magnitude, not precision.** "~40k RPS" is a great answer. "41,238 RPS" is a worse one — it's
  false precision that wastes time.
- **Round aggressively.** Use powers of 10 and clean fractions. 86,400 s/day → **~10⁵**. A month → **~2.5M s**.
- **Always attach the "so what."** After every number say: *"This matters because it means we need ___."*
- **Do it out loud.** The reasoning is the point; the number is a byproduct.

---

## Numbers to memorize (the whole table)

**Time**

| Quantity | Rounded value |
|---|---|
| Seconds/day | 86,400 ≈ **10⁵** |
| Seconds/month | ≈ **2.5 × 10⁶** |
| Seconds/year | ≈ **3 × 10⁷** (31.5M) |

**Powers of 10 → data sizes**

| Power | Name | Rough real-world |
|---|---|---|
| 10³ | KB | one long tweet |
| 10⁶ | MB | a small photo, a minute of MP3 |
| 10⁹ | GB | a movie (SD), RAM in a box |
| 10¹² | TB | a big disk |
| 10¹⁵ | PB | "web-scale" storage |

**Latency numbers every engineer should know** (rounded)

| Operation | Time | Takeaway |
|---|---|---|
| L1 cache reference | ~1 ns | — |
| Main memory (RAM) reference | ~100 ns | memory is ~100× L1 |
| Read 1 MB sequentially from RAM | ~10 µs | — |
| SSD random read | ~100 µs | ~1000× RAM |
| Round trip within a datacenter | ~0.5 ms | cross-service calls add up |
| Read 1 MB from SSD | ~1 ms | — |
| Disk (HDD) seek | ~10 ms | avoid on the hot path |
| Round trip **cross-region** (e.g. US↔EU) | ~100–150 ms | *this* is why multi-region is hard |

The one that wins interviews: **a cross-region round trip is ~100 ms.** It explains why you can't just
"add another region" for latency, why you replicate data close to users, and why strong consistency across
regions is expensive.

---

## The estimation recipe (5 steps)

### 1. Start from users
`DAU` (daily active users) is the anchor. If given MAU, assume **DAU ≈ 20–50% of MAU**. State your ratio.

### 2. Actions per user per day → average RPS

```text
avg RPS = (DAU × actions per user per day) / seconds per day
        = (DAU × actions) / 10⁵
```

The `/10⁵` shortcut (seconds/day) makes this a one-liner in your head.

### 3. Average → peak
Traffic isn't flat. Multiply by a **peak factor of 2–10×** (state which). A safe default: **peak ≈ 2–3× average**
for global consumer apps; higher for spiky/regional ones.

```text
peak RPS ≈ avg RPS × 3
```

### 4. Split reads vs writes
State the **read:write ratio** — it's often the single most design-shaping number.

- Read-heavy (100:1, e.g. URL shortener, feeds) → **cache + read replicas**, reads are the problem
- Write-heavy (e.g. metrics ingestion, logging) → **partitioning, batching, LSM stores**, writes are the problem

### 5. Storage & bandwidth

```text
storage/day  = writes/day × bytes per record
storage/year = storage/day × 365   (≈ ×400, round up)
bandwidth    = RPS × payload size
```

For media systems (video/images) bandwidth usually dwarfs everything → that's your CDN + egress cost story.

---

## Worked example — URL shortener

**Given / assumed:** 100M new URLs/month, read:write = 100:1.

**Writes:**
```text
100M / month ÷ 2.5M s/month ≈ 40 writes/sec (avg)
peak ≈ 40 × 3 ≈ 120 writes/sec
```
→ *So what?* 120 writes/sec is **tiny** — a single Postgres handles this easily. **No sharding for writes.**

**Reads:**
```text
100:1 ratio → ~4,000 reads/sec avg, ~12,000 peak
```
→ *So what?* 12k reads/sec against the DB is a lot of identical lookups. **This is why we add a cache** —
short-code→URL is a perfect cache-aside key, and a high hit ratio drops DB read load by ~10×.

**Storage:**
```text
500 bytes/record × 100M/month × 12 months ≈ 600 GB/year
```
→ *So what?* ~600 GB/year fits comfortably on one node for years. **Storage is not the constraint** — latency is.
The whole design is "make reads fast," not "store a lot."

**The lesson:** three quick calculations told us the architecture — cache-first, read-optimized, single
primary DB, no sharding. *That's* what estimation is for.

---

## Worked example — video platform (why bandwidth dominates)

**Given:** 1M DAU, each watches 5 videos/day, avg 10 MB delivered per video.

```text
data served/day = 1M × 5 × 10 MB = 50 TB/day
avg egress      = 50 TB / 10⁵ s ≈ 500 MB/s ≈ 4 Gbps (avg), ~12 Gbps peak
```
→ *So what?* Egress is enormous and continuous. **This is the CDN + egress-cost story** — you never serve
video from app servers; you push bytes to a CDN and your #1 cost line is CDN egress, not compute or storage.
Contrast with the URL shortener where bandwidth was a rounding error.

---

## Which numbers actually change the architecture

| Number | If it's high, you reach for… |
|---|---|
| **Read RPS** | cache, read replicas, CDN |
| **Write RPS** | batching, write-behind, partitioning/sharding, LSM stores |
| **Read:write ratio** | decides whether reads or writes are your problem |
| **Payload / bandwidth** | CDN, object storage, compression |
| **Storage/year** | partitioning, tiered storage (hot/cold), archival |
| **Concurrent connections** | connection pooling, WebSocket fan-out design, sticky sessions |
| **Fan-out** (followers, subscribers) | fan-out-on-write vs on-read decision |
| **p99 latency target** | cache, precompute, co-locate data, avoid cross-region on hot path |

If a number doesn't map to a lever in this table, you probably don't need to compute it.

---

## Common traps

- **Precision theatre** — 5 significant figures of a made-up input. Round.
- **Numbers with no "so what"** — computing storage you never reference.
- **Forgetting peak** — designing for average traffic then getting paged at launch.
- **Ignoring the read:write ratio** — optimizing writes on a read-heavy system (or vice-versa).
- **Bandwidth blindness** — for media, egress is usually the real story and the real cost.

---

## Interview wording

> "Let me estimate the order of magnitude before I design."
> "I'll assume DAU is about 20% of MAU — does that sound right?"
> "Seconds in a day is roughly 10⁵, so that's about 40 writes per second on average, call it 120 at peak."
> "That's small enough that a single database handles writes — I won't shard prematurely."
> "The read:write ratio is 100:1, so reads are the problem; that's what the cache is for."
> "Bandwidth dominates here, so the real design question is CDN strategy and egress cost."

→ Next: **[Staff-level thinking](staff-level-thinking.md)**
