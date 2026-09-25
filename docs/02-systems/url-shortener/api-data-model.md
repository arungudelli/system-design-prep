# URL Shortener — API & Data Model

## APIs

Keep it to the essentials. Two endpoints do the whole job.

### Create a short URL

```text
POST /v1/urls
Authorization: Bearer <token>        (if accounts exist; optional for anonymous)
Idempotency-Key: <uuid>              (optional but recommended — see below)

Request:
{
  "longUrl": "https://example.com/some/very/long/path?with=query",
  "customAlias": "my-brand",         // optional
  "ttlDays": 365                      // optional
}

201 Created:
{
  "shortCode": "aB3x9Zq",
  "shortUrl": "https://sho.rt/aB3x9Zq",
  "longUrl":  "https://example.com/...",
  "expiresAt": "2027-09-25T00:00:00Z" // null if no expiry
}

409 Conflict   — customAlias already taken
400 Bad Request — malformed / unsafe URL
```

### Resolve (redirect)

```text
GET /{shortCode}

302 Found
Location: https://example.com/some/very/long/path?with=query

404 Not Found  — unknown or expired code
410 Gone       — expired (optional, more precise than 404)
```

> **Redirect status matters:** use **302 (temporary)** so clients/CDNs don't cache the mapping and every click
> flows through us (needed for analytics and for the ability to change/disable a link). Use **301 (permanent)**
> only when you *want* browsers/CDNs to cache it and you don't need per-click tracking. This is a real
> trade-off — see [Trade-offs](tradeoffs.md) and [Deep Dives](deep-dives.md).

### (Nice-to-have) analytics read

```text
GET /v1/urls/{shortCode}/stats
200 { "clicks": 10432, "byDay": [...], "topReferrers": [...] }
```

---

## Idempotency (why the header matters)

`POST /urls` is a write that clients may retry (network blip, timeout). Without protection, a retry could
create **two codes for the same request**. Two clean options:

1. **Idempotency-Key**: client sends a UUID; server stores `key → shortCode` and returns the same code on
   replay. Best for "same request must yield same code."
2. **Deterministic mapping**: if the product says "same long URL always maps to the same code," a unique
   index on `longUrl` makes creation naturally idempotent (second insert returns the existing row).

Most shorteners *don't* dedupe by long URL (two users shortening the same URL usually want separate codes +
separate analytics), so prefer the **Idempotency-Key** approach for retry-safety while still allowing
distinct codes. See [idempotency pattern](../../01-patterns/idempotency.md) *(coming)*.

---

## REST vs gRPC

- **Public API + the redirect itself → REST/HTTP.** The redirect is literally an HTTP `GET` + `302`; it must
  work from any browser. No contest.
- **Internal** calls (e.g. API server → Key Generation Service) can be gRPC for typed, low-latency RPC, but
  at this scale plain HTTP is fine. Don't over-engineer.

---

## Data model

### Access patterns first (always)

1. **`shortCode → longUrl`** — the hot path, ~12k/s, must be <50 ms. *(point lookup by primary key)*
2. `create(longUrl) → shortCode` — ~120/s. *(insert)*
3. `customAlias` uniqueness check — occasional. *(point lookup / unique constraint)*
4. (optional) analytics append + aggregate — separate, async path.

Pattern #1 dominates and is a **pure key-value lookup by primary key**. That's the deciding fact.

### Schema

```text
Table / Collection: urls
-----------------------------------------------------------
short_code    VARCHAR(8)   PRIMARY KEY     -- the lookup key
long_url      TEXT         NOT NULL
created_at    TIMESTAMP    NOT NULL
expires_at    TIMESTAMP    NULL            -- null = never
owner_id      BIGINT       NULL            -- if accounts exist
-----------------------------------------------------------
```

- **Primary key = `short_code`.** The dominant query is a lookup by it — make it the PK so it's the clustering/
  hash key and reads are single-key fast.
- **Optional unique index on `long_url`** *only if* the product wants "one code per URL". Usually omit.
- **Index on `expires_at`** *only if* you actively sweep expired rows (or use the store's TTL feature instead).
- **`owner_id` index** if you offer per-user link listing.

### SQL or NoSQL? (see the full argument in Trade-offs)

Either works — the access pattern is a simple keyed lookup:

- **NoSQL KV (DynamoDB / Cassandra / Redis-backed):** the natural fit — `shortCode` as partition key gives
  O(1) lookup, effortless horizontal scale, and no relational features are needed. Great default at scale.
- **SQL (Postgres/MySQL):** perfectly fine at this scale (~120 writes/s, ~600 GB/yr). A single primary +
  read replicas + cache handles it for years, and you get transactions/constraints for free (handy for
  custom-alias uniqueness). Simpler to operate if the team already runs SQL.

> **Interview line:** *"The access pattern is a keyed lookup with no joins, so a KV store is the natural fit
> and scales trivially. But at this write volume a single SQL primary with replicas and a cache is also
> completely adequate and simpler to operate — I'd pick based on the team's existing stack and whether we
> expect to need relational features like alias constraints."*

---

## Key length math (why 7)

```text
base62 (A-Z, a-z, 0-9) → 62 symbols per position
length 6 → 62^6 ≈ 56 billion
length 7 → 62^7 ≈ 3.5 trillion
length 8 → 62^8 ≈ 218 trillion

Need ~6 billion codes over 5 years → 7 chars gives ~570× headroom.
```

**Choose 7.** Short enough to be user-friendly, large enough to never worry about exhaustion or (for random
schemes) collisions.

→ Next: **[Architecture](architecture.md)**
