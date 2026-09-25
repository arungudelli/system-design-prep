# URL Shortener — Deep Dives

The interview lives here. The high-level design is easy; the signal is in how you reason about these sub-problems.

---

## 1. Unique short-code generation (the core sub-problem)

Requirement: every code is **unique**, **short** (7 chars), generated **fast** (~120/s, no contention), and —
depending on the product — possibly **unguessable**.

### Option A — Hash the URL (e.g. MD5/SHA → take first 7 base62 chars)
```text
code = base62( hash(longUrl + salt) )[:7]
```
- ✅ Stateless, deterministic, no counter to coordinate.
- ❌ **Collisions** are inevitable when truncating a hash → you must check the DB and retry on collision
  (a read + conditional write on every create). At low write volume this is tolerable, but it adds a DB round-trip.
- ❌ Same URL → same code (unless salted per request), which may be undesirable for per-link analytics.
- **Verdict:** simple, but collision handling makes it fiddly. Fine as a fallback; not the cleanest.

### Option B — Global counter + base62 encode  ✅ (preferred)
```text
next_id = incrementGlobalCounter()   // 1, 2, 3, ...
code    = base62(next_id)            // "1" -> "1", 12345 -> "3d7", ...
```
- ✅ **No collisions ever** — each integer is unique by construction.
- ✅ Fast, no DB read on create (just the counter + insert).
- ❌ The global counter is a single point of contention/failure **if naive**.
- ❌ Codes are **sequential → guessable/enumerable** (mitigate below).

**Making the counter scale — the Key Generation Service (KGS):**
Instead of every create hitting one counter, a KGS **pre-allocates ranges** to each API server:

```text
KGS owns the counter. It leases ranges:
  API server A ← [1,000,000 .. 1,999,999]
  API server B ← [2,000,000 .. 2,999,999]
Each server increments its local range in memory, converting to base62.
When a server nears the end of its range, it leases the next range.
```
- ✅ **No coordination on the hot path** — servers mint codes from memory.
- ✅ No collisions (ranges are disjoint).
- ✅ KGS is cheap and easy to make HA (small state: "next range to hand out", persisted).
- ⚠️ **A server crash "loses" its unused range** — that's fine; ranges are cheap, we just skip those codes.
  (Never reuse them — reuse risks handing a recycled code a new target while old links still point to it.)

**Making counter codes unguessable (if required):**
Don't expose the raw sequential integer. Options: encode `permute(id)` with a reversible bijection (e.g.
Feistel/`XOR`+multiply mod 62⁷), or interleave a few random base62 chars. You keep collision-freedom while
breaking enumeration. Only do this if privacy is a stated requirement — otherwise sequential is fine.

### Option C — Random 7 chars + uniqueness check
```text
code = random 7 base62 chars; INSERT ... IF NOT EXISTS; retry on conflict
```
- ✅ Unguessable by default.
- ❌ Needs a uniqueness check per create; collision probability rises as the space fills (birthday paradox),
  though with 3.5T space and 6B used, collisions are rare for years.
- **Verdict:** good when unguessability matters and write volume is modest.

> **Interview answer:** *"I'd use a counter encoded in base62 via a Key Generation Service that leases ranges
> to each app server — that gives collision-free, contention-free 7-char codes. If codes must be unguessable,
> I'd run the counter through a reversible permutation so they're still unique but not enumerable. I'd avoid
> hash-truncation because collision handling adds a read-and-retry to every create."*

---

## 2. Caching strategy

- **Pattern: cache-aside** for reads (app checks cache, on miss reads DB and populates). Codes are
  **immutable** → cached values never go stale → we can use **long TTLs** and get a very high hit ratio.
- **On create: write-through** the new mapping into cache so a freshly created link is instantly hot (helps
  the common "create then immediately share" flow).
- **Eviction: LRU** — keep the hot set; cold links fall out and are re-fetched from DB on the rare click.
- **Size:** ~10 GB holds the hot set (see [Capacity](02-systems/url-shortener/capacity.md)) → 90%+ hit ratio.

### "What happens if Redis goes down?"  (they *will* ask)
- Reads **fall through to the DB / read replicas** — higher latency, still correct. The system **degrades, not
  fails**. This is why the DB + replicas must be sized to survive a cache outage (at least briefly).
- **Cache stampede risk on cold start / mass expiry:** when the cache is empty, a burst of misses can all hit
  the DB for the same hot key. Mitigate with **request coalescing / single-flight** (only one loader per key),
  **jittered TTLs** (so keys don't all expire together), and pre-warming the top-N on restart.
- **Cache penetration:** repeated lookups of **non-existent** codes bypass the cache and hammer the DB. Mitigate
  by caching negative results (`code → NOT_FOUND` with a short TTL) and/or a **Bloom filter** of known codes.

---

## 3. Redirect: 301 vs 302 (a genuine trade-off — expect a follow-up)

| | **301 Moved Permanently** | **302 Found (temporary)** |
|---|---|---|
| Browser/CDN caches it? | **Yes** — future clicks skip your server | **No** — every click hits your server |
| Analytics per click? | ❌ lost after first cache | ✅ you see every click |
| Can you change/disable the link later? | ❌ hard (clients cached it) | ✅ full control |
| Load on your servers | Lower (offloaded to clients/CDN) | Higher (all traffic flows through) |

> **Verdict / interview line:** *"I'd default to **302** so we keep click analytics and the ability to expire
> or change a link. I'd only use **301** for links where we explicitly want browser/CDN caching to offload
> traffic and we've accepted losing per-click tracking. It's a control-and-analytics vs. offload-and-latency
> trade-off, not a right/wrong choice."*

---

## 4. Custom aliases

- Client proposes `customAlias`; we must ensure it doesn't collide with generated codes **or** other aliases.
- **Uniqueness check:** a unique constraint on `short_code` handles it — insert fails → return `409 Conflict`.
- **Namespace collision:** generated codes and custom aliases share the same key column, so a `INSERT ... IF
  NOT EXISTS` naturally arbitrates. To avoid a custom alias ever colliding with a *future* generated code,
  either (a) reserve a separate prefix/length for custom aliases, or (b) since generated codes come from the
  counter space, just let the unique constraint reject the rare clash.
- **Abuse:** custom aliases enable brand impersonation/squatting → may need moderation/reserved words.

---

## 5. Analytics (nice-to-have, but know the shape)

- **Never on the critical path.** On redirect, emit a lightweight **click event** (code, timestamp, referrer,
  geo/IP, UA) to a **queue** (Kafka/SQS); consumers aggregate into counters / a data warehouse asynchronously.
- Counting is **eventually consistent** — a slightly stale click count is fine. Use approximate structures
  (e.g. HyperLogLog for unique visitors) if exact counts aren't required.
- This is the concrete reason to prefer **302**: a cached **301** means most clicks never reach you, so you
  can't count them.

---

## 6. Expiry / TTL

- Store `expires_at`; on read, if expired → `404`/`410` and (lazily) delete.
- **Active cleanup:** a background sweeper deletes expired rows, or use the datastore's native TTL (DynamoDB
  TTL, Redis TTL) to auto-expire. Lazy + background combined keeps the table lean without a heavy cron.
- Expiry lets you **reclaim key space** — relevant only if you're near exhaustion (we're not, with 7 chars).

---

## 7. Abuse & safety (right-sized)

- **Malicious URLs:** run a safe-browsing / blocklist check on **create** (and optionally re-check on resolve,
  since a URL can turn malicious later). Short URLs are a classic phishing vector — mention this; don't build a
  whole threat platform.
- **Rate limiting** on `POST /urls` per IP/account to stop bulk abuse. See [rate limiting](01-patterns/rate-limiting.md) *(coming)*.
- **Open-redirect hygiene:** validate the target scheme (only http/https), and consider an interstitial warning
  page for untrusted targets.

→ Next: **[Trade-offs](02-systems/url-shortener/tradeoffs.md)**
