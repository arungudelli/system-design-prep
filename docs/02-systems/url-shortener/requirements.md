# URL Shortener — Requirements

## 1. Problem statement (the interview prompt)

> "Design a URL shortening service like TinyURL or Bit.ly. Users give us a long URL and we return a short
> one. When someone visits the short URL, we redirect them to the original."

Deliberately vague — that's the test. Don't start drawing. Start clarifying.

---

## 2. Clarifying questions

Ask these, and know *why each one changes the design*.

### Product
- **Who creates short URLs — anonymous users, or authenticated accounts?**
  *Why:* auth enables per-user quotas, ownership, and analytics dashboards; anonymous means heavier abuse control.
- **Do we need custom aliases** (e.g. `short.ly/my-brand`)?
  *Why:* changes key generation — custom aliases need a uniqueness check and collide with generated codes.
- **Do links expire?** Default TTL, or forever?
  *Why:* expiry lets us reclaim keys and bound storage; "forever" changes the key-space and cleanup story.
- **Do we need click analytics** (counts, geo, referrer)?
  *Why:* analytics conflicts with aggressive redirect caching (301) — it forces every click through us (302).

### Scale
- **How many new URLs per month?** **Read:write ratio?**
  *Why:* this single ratio decides whether reads or writes are the bottleneck (it's reads).
- **Any single "viral" link that gets a huge share of traffic?**
  *Why:* that's a **hot key** — one cache entry taking massive load; changes cache design.

### Consistency
- **Is it OK if a just-created short URL takes a second to become resolvable?**
  *Why:* if eventual consistency is fine, we can cache/replicate freely; if read-your-writes is required, we route reads carefully.
- **Must a code map to exactly one URL, forever (immutable)?**
  *Why:* immutability makes caching trivially safe (a code's target never changes → cache forever).

### Latency
- **What's the acceptable redirect latency?**
  *Why:* a redirect is on the user's critical path; "p99 < 50 ms" pushes us to cache-first + edge.

### Availability
- **What's worse: being unable to *create* a link, or unable to *resolve* one?**
  *Why:* resolution (reads) is the mission-critical path; creation can tolerate brief downtime. This tells us where to spend availability budget.

### Security / compliance
- **Do we need to block malicious/phishing URLs?**
  *Why:* short URLs are a classic phishing vector → may need a safe-browsing check on create and/or resolve.
- **Should short codes be unguessable** (no enumeration of others' links)?
  *Why:* sequential counter codes are enumerable; if privacy matters we add randomization/hashing.

---

## 3. Functional requirements

### Must have
1. **Create** a short code for a given long URL: `POST /urls` → `shortCode`.
2. **Redirect**: `GET /{shortCode}` → HTTP redirect to the original long URL.
3. **Uniqueness**: every short code maps to exactly one long URL; no collisions.

### Nice to have (name them, then defer)
- Custom aliases
- Link expiration / TTL
- Click analytics (counts, geo, referrer, device)
- User accounts + dashboard + per-user link management
- Malicious-URL detection

> **In an interview, say:** *"Must-haves are create, redirect, and uniqueness. I'll defer custom aliases,
> analytics, and expiry — each is a clean add-on I can revisit if we have time. This keeps scope tight."*

---

## 4. Non-functional requirements (quantified)

| NFR | Target | Why |
|---|---|---|
| **Redirect latency** | p99 **< 50 ms** server-side | It's on the user's critical path — the whole product is "click → arrive". |
| **Availability (reads)** | **99.99%** | If redirects fail, every existing link is broken — catastrophic. |
| **Availability (writes)** | 99.9% is fine | Failing to create a new link briefly is annoying, not catastrophic. |
| **Durability** | Must **never lose** a created mapping | A lost mapping = a permanently dead link. |
| **Consistency** | **Eventual** is acceptable for reads; codes are **immutable** | A code never changes target, so caching is safe; a ~second of propagation delay on create is fine. |
| **Scalability** | Handle 100:1 read:write, scale reads horizontally | Reads are the growth axis. |
| **Uniqueness** | 100% — no two long URLs share a code accidentally | Correctness requirement. |

**Dominant NFRs:** low redirect latency + high read availability. Everything in the design serves those two.

---

## 5. What we are explicitly NOT building (this pass)

- Full analytics pipeline (mention how it'd attach, don't build it)
- Multi-region active-active (single region + multi-AZ is enough at this scale; note when we'd revisit)
- A general link-management product (folders, teams, billing)

→ Next: **[Capacity](capacity.md)**
