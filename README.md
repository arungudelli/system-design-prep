# 🧭 System Design Prep

> Personal, incrementally-built prep for **FAANG Staff / Principal / EM** system-design interviews.
> The goal is **not** to memorize architectures — it's to *derive* them from requirements, recognize
> reusable patterns, reason about trade-offs, find bottlenecks, and defend decisions like a Staff Engineer.

📖 **Read it as a website:** [arungudelli.github.io/system-design-prep](https://arungudelli.github.io/system-design-prep)
(rendered from this repo's Markdown via [Docsify](https://docsify.js.org) — zero build, GitHub Pages)

---

## How to use this repo

Every system is worked through the **same 20-step framework** so the *method* becomes muscle memory.
For each topic the loop is:

> teach the concept → simplest architecture → increase scale → challenge the design → answer follow-ups → critique → capture lessons

**Always start simple.** No Kafka / Kubernetes / Cassandra / sharding / microservices until a requirement
forces it — and every added component must answer *"why do we need this, and what does it solve that the
simpler design can't?"*

---

## Repository map

| Section | What's in it | Status |
|---|---|---|
| [`00-framework/`](00-framework/system-design-framework.md) | The 20-step method, capacity math, staff-level thinking, interview wording, checklist | 🟢 done |
| [`01-patterns/`](01-patterns/caching.md) | **Caching 🟢 · Sharding 🟢 · Queues & Workers 🟢 · Idempotency 🟢 · Rate Limiting 🟢** · realtime, object storage, observability, multi-region (planned) | 🟢 building |
| [`02-systems/`](02-systems/url-shortener/README.md) | **URL Shortener 🟢 · Web Crawler 🟢 · Job Scheduler 🟢** · notifications, chat, file storage, YouTube, rate limiter, metrics, news feed, payments, ride-sharing (planned) | 🟢 building |
| `03-tradeoffs/` | SQL vs NoSQL, Kafka vs SQS, sync vs async, push vs pull, WebSocket vs SSE, active-active vs active-passive | ⚪ planned |
| `04-interview-drills/` | Bottleneck drills, failure scenarios, scalability & DB questions, staff-level follow-ups | ⚪ planned |
| `05-cheatsheets/` | 1–2 page revision sheets | ⚪ planned |
| `06-mocks/` | Timed mock interviews | ⚪ planned |

---

## Progress

Building **one topic at a time** (per the learning plan in `prompt.txt`):

- [x] **00 · Framework** — the lens every design uses
- [x] **02 · URL Shortener** — first full per-system walkthrough
- [x] **01 · Caching pattern** — the highest-leverage read-scaling move
- [x] **01 · Partitioning & Sharding pattern** — the write/storage-scaling last resort
- [x] **01 · Queues & Workers pattern** — async processing foundation
- [x] **01 · Idempotency pattern** — what makes retries safe
- [x] **02 · Web Crawler** — assembles queue+workers, idempotency, sharding, caching
- [x] **01 · Rate Limiting pattern** — token/leaky bucket, windows, distributed Redis limiter
- [x] **02 · Distributed Job Scheduler** — scheduling, exactly-once effect, leases + fencing
- [ ] 01 · More patterns (realtime, observability, object storage…)
- [ ] 02 · Notification System
- [ ] 02 · Chat / Messaging
- [ ] 02 · File Storage (Dropbox)
- [ ] 02 · YouTube / Video
- [ ] 02 · Rate Limiter
- [ ] 02 · Metrics / Observability Platform
- [ ] 02 · News Feed
- [ ] 02 · Payment System
- [ ] 02 · Ride Sharing

---

## Start here

1. **[The 20-step framework](00-framework/system-design-framework.md)** — the backbone of every answer
2. **[Capacity estimation](00-framework/capacity-estimation.md)** — the only math you need, and *which numbers matter*
3. **[Staff-level thinking](00-framework/staff-level-thinking.md)** — the questions that separate senior from staff
4. **[Interview wording](00-framework/interview-wording.md)** — phrases that sound clear and experienced
5. **[Interview checklist](00-framework/interview-checklist.md)** — the fast pre-interview scan

---

<sub>Built with Claude Code · rendered with Docsify · this is a living document — notes get sharper after every mock.</sub>
