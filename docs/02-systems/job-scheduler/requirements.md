# Distributed Job Scheduler — Requirements

## 1. Problem statement (the interview prompt)

> "Design a distributed job scheduler. Users can schedule jobs to run at a specific time, on a recurring schedule
> (like cron), or as soon as possible. The system executes them reliably across many workers, retrying on failure."

Vague on purpose. The **execution guarantee** and **job types** massively shape the design — clarify first.

---

## 2. Clarifying questions

### Product / semantics
- **What kinds of jobs?** One-off "run at time T", recurring/cron, and/or immediate background jobs?
  *Why:* recurring adds next-fire computation + timezone/DST + catch-up; one-off is simpler.
- **What does a "job" do?** Run arbitrary user code? Call a webhook/HTTP endpoint? Enqueue to another system?
  *Why:* running untrusted code needs sandboxing/isolation (a huge concern); calling a URL is far simpler.
- **What execution guarantee?** At-least-once (may run twice), at-most-once (may skip), or "exactly-once"?
  *Why:* this is *the* central decision — it drives idempotency, leases, and how we handle crashes.
- **How precise must timing be?** Sub-second? Within a minute? Best-effort?
  *Why:* second-precision at scale needs a tighter fire loop + more careful clock handling than "within a minute."
- **What if the system was down at the fire time — run late (catch-up) or skip?**
  *Why:* changes whether we replay missed fires or drop them.

### Scale
- **How many scheduled jobs total? How many fire per second at peak?**
  *Why:* drives the due-scan strategy (index vs bucketed) and worker count.
- **Job duration?** Milliseconds, or long-running (minutes/hours)?
  *Why:* long jobs need lease extension/heartbeats so they aren't declared dead and re-run.

### Consistency / correctness
- **Is it worse to run a job twice or to miss it?**
  *Why:* payment charge → never twice (at-most-once-ish + idempotency); a reminder email → better twice than never.

### Availability / operational
- **Is missing a fire acceptable during a deploy/outage?** Priorities/fairness across tenants?
  *Why:* multi-tenant fairness + HA requirements.

### Security
- **Multi-tenant?** Isolation between tenants' jobs; per-tenant quotas?
  *Why:* noisy-neighbor + a runaway tenant shouldn't starve others.

---

## 3. Functional requirements

### Must have
1. **Schedule a job** — one-off (run at T) and recurring (cron/interval).
2. **Execute** the job at (approximately) the scheduled time via a worker.
3. **Reliability** — a job that's due **runs** (at-least-once), surviving worker/scheduler crashes.
4. **Retries** — failed jobs retry with backoff; give up to a DLQ after N attempts.
5. **Cancel / update** a scheduled job; **query** job status/history.

### Nice to have (name, then defer)
- **Exactly-once effect** (idempotency) — mention as the guarantee strategy; core, but call it out explicitly.
- **Job dependencies / DAGs** (Airflow-style) — flag as a big extension; defer.
- **Priorities & fairness**, per-tenant quotas.
- **Catch-up / backfill** of missed runs.
- **Sandboxed arbitrary code execution** (vs simple webhook) — flag the security cost.

> **Interview line:** *"Must-haves: schedule one-off + recurring jobs, execute near the scheduled time reliably with
> retries, and cancel/query. I'll treat exactly-once as an idempotency strategy I'll design in, and defer DAGs,
> priorities, and sandboxed code execution unless we have time."*

---

## 4. Non-functional requirements (quantified)

| NFR | Target | Why |
|---|---|---|
| **Timing accuracy** | fire within, e.g., **1s** of scheduled time (state your target) | Late fires may violate the product promise. |
| **Reliability** | a due job **runs at least once**; no silent drops | The whole point — scheduled work must happen. |
| **Execution guarantee** | **exactly-once *effect*** (at-least-once + idempotent) | Can't guarantee exactly-once delivery; make duplicates harmless. |
| **Durability** | scheduled jobs survive crashes/restarts | Losing the schedule = missed work. |
| **Scalability** | millions of scheduled jobs; scale fires/sec by adding workers | Grows with usage. |
| **Availability** | scheduler HA; a node dying doesn't stop firing | Continuous system. |
| **Isolation** | one tenant/job can't starve others; a hung job can't block the queue | Multi-tenant safety. |

**Dominant NFRs:** **reliable, near-on-time execution** + **exactly-once effect** + **durability of the schedule**.

---

## 5. What we are explicitly NOT building (this pass)

- **DAG / workflow orchestration** (dependencies between jobs) — that's Airflow/Step-Functions territory; note it.
- **Sandboxed arbitrary-code execution** — assume jobs are a webhook call or a known task type; flag the security
  cost of running untrusted code.
- The **downstream systems** the jobs act on (we trigger them; they're separate).

→ Next: **[Capacity](capacity.md)**
