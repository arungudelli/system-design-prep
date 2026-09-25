# URL Shortener — Capacity Estimation

> See the method in **[Capacity estimation](00-framework/capacity-estimation.md)**. The goal here is to
> produce the 3–4 numbers that decide the architecture — and to *use* each one.

---

## Assumptions (state these out loud)

- **100M new URLs / month** (new writes)
- **Read:write ratio = 100:1** (people click far more than they create)
- **Peak factor ≈ 3×** average
- **~500 bytes** stored per record (long URL + code + metadata)
- Links effectively **long-lived** (assume kept for years)

---

## Writes (create QPS)

```text
100M / month ÷ 2.5M seconds/month ≈ 40 writes/sec  (average)
peak ≈ 40 × 3                     ≈ 120 writes/sec
```

→ **So what?** 120 writes/sec is *tiny*. A single relational or KV node handles this without breaking a
sweat. **No write sharding, no fancy write path.** This kills the instinct to over-engineer.

---

## Reads (redirect QPS)

```text
reads ≈ 100 × writes
avg  ≈ 4,000 reads/sec
peak ≈ 12,000 reads/sec
```

→ **So what?** 12k reads/sec of *identical key lookups* is the real workload. Hitting the DB for every one
is wasteful and risks latency. **This is the entire justification for a cache** — `shortCode → longUrl` is a
perfect cache-aside key, and because codes are immutable, entries can live a long time → very high hit ratio.

---

## Storage

```text
per record ≈ 500 bytes
per year   ≈ 100M/month × 12 × 500 B ≈ 600 GB/year
5 years    ≈ 3 TB
```

→ **So what?** ~600 GB/year fits on a single node for years. **Storage is not the constraint.** We're not
partitioning for capacity — if we ever partition, it'll be for read/write *throughput* or blast-radius, not size.

---

## Cache sizing (the 80/20 rule)

A small fraction of links get most of the clicks. Cache the **hot set**, not everything.

```text
Assume 20% of links drive 80% of reads.
Cache ~100M hot entries × ~100 bytes (code + URL) ≈ 10 GB
```

→ **So what?** ~10 GB of Redis comfortably holds the hot set → **90%+ cache hit ratio** is realistic. That
drops DB read load from 12k/s to ~1k/s — now even a single primary + a replica is plenty.

---

## Bandwidth

Redirects are tiny (a `302` with a `Location` header — a few hundred bytes). At 12k/s that's a few MB/s.
→ **So what?** **Bandwidth is a non-issue** here (contrast with a video platform where it dominates). Don't
spend interview time on it beyond noting it's negligible.

---

## Key space (do this — it decides code length)

```text
base62 = [A-Z a-z 0-9] = 62 symbols
62^6 ≈ 56 billion
62^7 ≈ 3.5 trillion
62^8 ≈ 218 trillion
```

We create ~100M/month ≈ 1.2B/year ≈ 6B in 5 years.
→ **So what?** **7 characters (3.5T) is comfortable** — years of headroom with room for churn. 6 chars (56B)
would work for ~5 years but leaves little margin; 8 chars is future-proof but longer. **Pick 7.** (See the
key-length math in [API & Data Model](02-systems/url-shortener/api-data-model.md).)

---

## Summary — what the numbers told us

| Number | Value | Design consequence |
|---|---|---|
| Write QPS | ~120 peak | Single DB for writes; **no sharding** |
| Read QPS | ~12k peak | **Cache-first**, read replicas |
| Storage/year | ~600 GB | Fits one node for years; not a driver |
| Cache size | ~10 GB | 90%+ hit ratio feasible → DB barely touched |
| Bandwidth | few MB/s | Negligible |
| Key length | **7 base62 chars** | 3.5T space, ample headroom |

The whole architecture falls out of this: **read-optimized, cache-first, single-primary DB, 7-char keys.**

→ Next: **[API & Data Model](02-systems/url-shortener/api-data-model.md)**
