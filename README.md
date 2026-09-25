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
| `01-patterns/` | Caching, queues/workers, sharding, idempotency, rate limiting, realtime, object storage, observability, multi-region… | ⚪ planned |
| [`02-systems/`](02-systems/url-shortener/README.md) | **URL Shortener 🟢** · notifications, chat, web crawler, file storage, YouTube, rate limiter, job scheduler, metrics, news feed, payments, ride-sharing (planned) | 🟢 building |
| `03-tradeoffs/` | SQL vs NoSQL, Kafka vs SQS, sync vs async, push vs pull, WebSocket vs SSE, active-active vs active-passive | ⚪ planned |
| `04-interview-drills/` | Bottleneck drills, failure scenarios, scalability & DB questions, staff-level follow-ups | ⚪ planned |
| `05-cheatsheets/` | 1–2 page revision sheets | ⚪ planned |
| `06-mocks/` | Timed mock interviews | ⚪ planned |

---

## Progress

Building **one topic at a time** (per the learning plan in `prompt.txt`):

- [x] **00 · Framework** — the lens every design uses
- [x] **02 · URL Shortener** — first full per-system walkthrough
- [ ] 01 · Reusable patterns
- [ ] 02 · Web Crawler
- [ ] 02 · Distributed Job Scheduler
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
