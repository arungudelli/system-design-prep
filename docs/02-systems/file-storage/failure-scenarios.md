# File Storage & Sync — Failure Scenarios

> Themes: **never lose or corrupt a file** (durability is the whole point), **commit is the atomic truth** (uploaded
> blocks are nothing until committed), **retries are safe** (content-addressed + idempotent), and **a missed notification
> never means a missed change** (the sync poll is the backstop).

---

## Upload interrupted mid-transfer

- **Impact:** some chunks uploaded, some not; the file isn't fully there.
- **Contain:** **nothing is visible until the commit** — partial chunks are just orphan bytes. The client **re-runs
  `status()`** on reconnect and uploads **only the still-missing chunks** (resumable), then commits. No corruption, no
  wasted re-upload of what already landed. See [upload protocol](api-data-model.md#the-upload-protocol-chunk--check--upload--commit).

## Commit fails or times out

- **Impact:** blocks are up but the metadata transaction didn't confirm.
- **Contain:** the commit is **idempotent** on `(path, chunks, baseVersion)` — the client retries safely; either it
  already succeeded (return the same version) or it applies now. Orphan blocks from abandoned attempts are reclaimed by
  **refcount GC**. Better a retry than a half-written tree.

## Two devices edit the same file offline (conflict)

- **Impact:** divergent edits; risk of silent data loss.
- **Contain:** **baseVersion detects divergence** at commit; the server **keeps both** (a conflicted copy) and syncs both
  everywhere. Never last-writer-wins by default. See [conflicts](deep-dives.md#4-conflict-resolution-never-silently-lose-data).

## Chunk corruption (bit rot / bad transfer)

- **Impact:** a chunk's bytes no longer match its hash.
- **Contain:** the **hash is an integrity check** — verify `hash(bytes)==address` on read; a mismatch ⇒ refetch from
  another replica/origin. The object store's **erasure coding + replication** repairs bit rot underneath. Content
  addressing makes corruption **detectable**, not silent.

## Metadata service shard down / slow

- **Impact:** can't commit or list files in the affected namespaces.
- **Contain:** **sharded by namespace** → blast radius is those namespaces, not all users. Failover to a replica;
  reads may serve from a replica (consistent within the shard). Uploads can still stage blocks; commits queue/retry until
  the shard returns. Degrade to **read-only / delayed sync**, never data loss.

## Block store / region outage

- **Impact:** can't fetch (or store) some chunks.
- **Contain:** object store is **multi-AZ replicated / erasure-coded** (11 nines) → an AZ loss is transparent; a region
  loss fails over to a replicated region. Downloads fall back from CDN to origin/replica. Accepted (committed) data is
  durable by construction.

## Notification service down / missed push

- **Impact:** a device isn't told to pull → could look stale.
- **Contain:** notifications are an **optimization, not truth** — the client's **periodic cursor-based `/sync` poll**
  catches every change regardless. Worst case sync latency degrades from seconds to the poll interval. See
  [notification loop](deep-dives.md#5-the-notification--sync-loop-near-real-time-cheaply).

## Thundering-herd sync (mass reconnect / popular shared folder)

- **Impact:** many devices call `/sync` + download at once (e.g. a shared folder update to 10k members, or a region
  reconnect).
- **Contain:** **backoff + jitter** on clients; **CDN** absorbs the block downloads (same chunks, cached); notification
  fan-out sends a signal, spreading the actual pulls over time. Metadata reads are cached.

## Duplicate uploads / double commit

- **Impact:** the same content or commit submitted twice (retries, races).
- **Contain:** blocks are **content-addressed** (PUT same hash = no-op) and commits are **idempotent** → duplicates
  collapse naturally. Dedup means re-uploading identical content stores nothing new.

## Reference-counting race → premature GC

- **Impact:** a chunk hits refcount 0 and is deleted while a concurrent upload is about to reference it → data loss.
- **Contain:** GC is **conservative** — async mark-and-sweep with a **grace period**, never delete on the hot path;
  re-check refcount at sweep time. Delete **late** rather than early. See [GC](deep-dives.md#7-garbage-collection-trillions-of-immutable-chunks).

## Client clock skew / stale cursor

- **Impact:** a client with an old cursor or wrong clock could mis-order.
- **Contain:** ordering comes from the **server-assigned monotonic namespace cursor/version**, not client clocks; the
  client just replays "everything after my cursor." Server time / version is authoritative.

## Malicious client (bad hashes, quota abuse)

- **Impact:** a client claims a hash it didn't upload, or spams uploads.
- **Contain:** server **verifies `hash(bytes)`** on upload (can't claim content you don't have — the basis for
  proof-of-possession in dedup); **per-user quotas + rate limits** ([rate limiting](../../01-patterns/rate-limiting.md))
  bound abuse; ACL checks on every metadata op.

---

## Failure-handling toolkit used here

`commit-is-atomic-truth` (partial uploads are invisible orphans) · `content-addressed + idempotent` (safe retries, no
dup) · `hash = integrity check` (detect corruption) · `namespace sharding` (blast-radius isolation) · `object-store
erasure coding / replication` (durability) · `notification = optimization, sync poll = backstop` · `baseVersion conflict
detection + keep-both` (no silent loss) · `conservative refcount GC with grace period` · `CDN + backoff/jitter`
(thundering herd) · `server verifies hashes + quotas` (malicious clients).

## Priorities (say this)

> *"My one non-negotiable: never lose or corrupt a file. Everything serves that — the commit is the atomic unit, so a
> partial or interrupted upload is just invisible orphan blocks I GC later; content addressing makes retries idempotent
> and corruption detectable; conflicts keep both copies rather than overwriting; and the object store's erasure coding
> gives 11-nines durability. Availability degrades gracefully — read-only or delayed sync during a shard outage, and a
> missed notification just falls back to the periodic sync poll — but committed data is never lost."*

→ Next: **[Interview Questions](interview-questions.md)**
