# Notification System — Requirements

## 1. Problem statement (the interview prompt)

> "Design a notification system. Other services trigger notifications (order shipped, OTP, friend request,
> marketing campaign) and we deliver them to users across push, SMS, email, and in-app — at scale, respecting
> user preferences."

Vague on purpose. **Which channels, transactional vs bulk, and delivery guarantees** reshape the design — clarify.

---

## 2. Clarifying questions

### Product
- **Which channels?** Push (mobile/web), SMS, email, in-app feed? Each has a different provider + constraints.
  *Why:* channels differ wildly in latency, cost, throughput limits, and reliability.
- **Transactional, bulk/marketing, or both?** OTP/receipts vs campaigns.
  *Why:* transactional is low-latency, high-priority, must-deliver; bulk is high-volume, latency-tolerant, opt-out-
  governed. They need **separate lanes** and different guarantees.
- **Who triggers notifications?** Many internal services via an API/event bus, or a few?
  *Why:* defines the ingestion contract and fan-out.
- **User preferences / opt-out / quiet hours?** Per-channel, per-category?
  *Why:* must-have for product + legal (CAN-SPAM/TCPA/GDPR); checked before every send.
- **Templates & localization?** Do we render templates, support multiple languages?
  *Why:* a template service + i18n is a real component.

### Scale
- **How many notifications/day? Peak/sec?** Biggest fan-out (a broadcast to all users)?
  *Why:* fan-out amplification (1 event → millions of sends) is the core scaling problem.

### Consistency / delivery guarantee
- **At-least-once (may duplicate) or best-effort?** Is a duplicate OTP acceptable?
  *Why:* drives dedup + idempotency. Usually at-least-once + dedup; a duplicate is annoying, a missed OTP is worse.
- **Do we need read receipts / delivery status?**
  *Why:* delivery tracking is a large extra write path.

### Latency
- **How fast must a transactional notification arrive (OTP)?** Seconds?
  *Why:* OTP p99 in seconds needs a priority lane that bypasses bulk backlogs.

### Security / compliance
- **PII handling, opt-out enforcement, regional rules (TCPA for SMS, GDPR)?**
  *Why:* legal + trust; opt-out must be honored and auditable.

---

## 3. Functional requirements

### Must have
1. **Accept a notification request** from internal services (event + recipient(s) + template + data).
2. **Multi-channel delivery** — push, SMS, email (in-app optional first).
3. **User preferences** — respect per-user, per-channel, per-category opt-in/out (and quiet hours).
4. **Reliability** — at-least-once delivery with **retries** and a **DLQ**; don't lose or double-send (dedup).
5. **Provider integration** — send via external providers with failover.

### Nice to have (name, then defer)
- **Delivery/read tracking** & analytics (mention; large write path).
- **Templates + localization** (design the hook; can defer detail).
- **Prioritization / rate limiting per user** (transactional vs bulk lanes) — important; call it out.
- **Scheduling** (send later) — delegate to the [job scheduler](02-systems/job-scheduler/README.md).
- **Aggregation/digest** ("3 people liked your post" instead of 3 notifications).

> **Interview line:** *"Must-haves: accept requests, deliver across push/SMS/email, honor preferences, and be
> reliable with retries/DLQ/dedup. I'll design hooks for templates, delivery tracking, and priority lanes, and
> defer digests and scheduling (the latter to a job scheduler) unless we have time."*

---

## 4. Non-functional requirements (quantified)

| NFR | Target | Why |
|---|---|---|
| **Transactional latency** | OTP/receipt delivered **p99 < a few seconds** to the provider | Users wait on OTPs; must beat bulk. |
| **Throughput** | millions/day; handle **large fan-out** (broadcast to all users) | Campaigns + platform events. |
| **Reliability** | **at-least-once**, no silent drops; **dedup** so no double-send | Missed OTP is bad; duplicate is annoying. |
| **Availability** | ingestion always accepts (buffer if downstream down) | Producers shouldn't fail because a provider is down. |
| **Preference correctness** | never send what a user opted out of / during quiet hours | Product + legal. |
| **Provider resilience** | survive a provider outage/throttle; fail over | Third parties are unreliable. |
| **Cost** | SMS is expensive; email cheap; push cheap | Channel cost differs by ~1000×. |

**Dominant NFRs:** **reliable multi-channel delivery** + **fan-out scale** + **priority/preference correctness** +
**resilience to flaky providers**.

---

## 5. What we are explicitly NOT building (this pass)

- The **event sources** (they call us) and the **providers** (we call them) — we own the middle.
- **Scheduling** (send-at-time) → delegate to the [job scheduler](02-systems/job-scheduler/README.md); note the seam.
- Full **analytics/warehouse** for delivery data — note the pipeline, don't build it.

→ Next: **[Capacity](02-systems/notification-system/capacity.md)**
