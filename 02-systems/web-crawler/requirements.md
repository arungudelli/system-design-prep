# Web Crawler — Requirements

## 1. Problem statement (the interview prompt)

> "Design a web crawler like the one that powers a search engine. It starts from seed URLs, downloads pages,
> extracts links, and follows them to discover and download more of the web. It should refresh pages over time."

Vague on purpose — the **purpose** of the crawler massively changes the design. Clarify before designing.

---

## 2. Clarifying questions

### Product / purpose
- **What is the crawl *for*?** Search indexing? A specific dataset (e.g. price monitoring, ML training corpus)?
  Archiving? Malware scanning?
  *Why:* purpose sets scope, prioritization, freshness needs, and what we extract/store. "Search index" =
  broad + prioritized + refreshed; "price monitor" = narrow + high-freshness on a few sites.
- **HTML only, or also render JavaScript (SPA sites)?**
  *Why:* rendering needs a headless browser per fetch → 10–100× the CPU/memory and cost. Huge design fork.
- **Text/HTML only, or also images/PDF/video?**
  *Why:* changes bandwidth, storage, and parsing pipeline dramatically.
- **How fresh must content be?** Recrawl cadence — hourly? daily? monthly?
  *Why:* freshness turns a one-shot crawl into a continuous, prioritized re-crawl scheduler.

### Scale
- **How many pages total / per month?** How many domains?
  *Why:* pages/s, storage, bandwidth, fetcher count all flow from this. Billions of pages is a very different
  machine than millions.
- **Politeness constraints?** Max requests/sec per domain?
  *Why:* politeness caps throughput per host — you can't just add fetchers to go faster on one site.

### Consistency / correctness
- **Is it OK to occasionally fetch the same URL twice, or must dedup be strict?**
  *Why:* strict dedup needs coordinated "seen" state; loose dedup is cheaper (Bloom filter with rare false-neg refetch).
- **Do we need the link graph** (who links to whom), or just page content?
  *Why:* the graph (for PageRank-style ranking) is a large extra dataset.

### Availability / operational
- **Is this continuous (always running) or a batch job?**
  *Why:* continuous → checkpointing, resumability, long-lived frontier; batch → simpler.

### Security / compliance / etiquette
- **Must we obey `robots.txt` and crawl-delay?** (Almost always **yes**.)
  *Why:* it's the law of the land for crawlers; ignoring it gets you blocked/blacklisted/sued.
- **Any domains to include/exclude?** Respect `noindex`/`nofollow`?
  *Why:* scope + compliance.

---

## 3. Functional requirements

### Must have
1. **Seed & fetch:** start from seed URLs, download page content over HTTP(S).
2. **Extract & follow:** parse pages, extract links, enqueue new/unseen URLs.
3. **Deduplicate URLs:** never (re)crawl the same URL unnecessarily.
4. **Politeness:** obey `robots.txt`, limit rate/concurrency per domain, identify via User-Agent.
5. **Store content:** persist fetched pages for downstream consumers (indexer, etc.).

### Nice to have (name, then defer)
- Content **deduplication** (near-duplicate detection) — mention it's important; can defer the SimHash detail.
- **Freshness / recrawl** scheduling.
- **JavaScript rendering** (headless browser) — flag the cost, usually defer.
- **Priority crawling** (crawl important pages first).
- Link-graph construction, non-HTML media, distributed multi-region crawling.

> **Interview line:** *"Must-haves are fetch, extract-and-follow, URL dedup, politeness, and storage. I'll treat
> content dedup, freshness, JS rendering, and prioritization as extensions — I'll design hooks for them but defer
> the details unless we have time."*

---

## 4. Non-functional requirements (quantified)

| NFR | Target | Why |
|---|---|---|
| **Throughput** | ~**400+ pages/s** sustained (scale to thousands) | It's the whole point — coverage over time. |
| **Politeness** | ≤ 1 in-flight request per host by default; obey crawl-delay | Hard external constraint; violate it and you get blocked. |
| **Scalability** | Horizontal — add fetchers/frontier shards to crawl more | The web is huge and grows. |
| **Robustness** | Survive traps, malformed HTML, slow/hostile servers, crashes; **resumable** | The web is adversarial and unreliable. |
| **Freshness** | Recrawl high-value pages on a cadence (e.g. news hourly, static monthly) | Stale index = bad search. |
| **Extensibility** | Pluggable parsers/extractors | New content types over time. |
| **Cost** | Bandwidth + storage are the big drivers | Billions of pages = real money. |
| **Availability** | Continuous; a fetcher dying must not lose queued work | Long-running system. |

**Dominant NFRs:** **throughput at scale** + **politeness** (they're in tension — you maximize global throughput
*subject to* per-host politeness) + **robustness/resumability**.

---

## 5. What we are explicitly NOT building (this pass)

- The **search index / ranking** itself (the crawler *feeds* it; PageRank/indexing is a separate system).
- **JS rendering** at first (note the 10–100× cost; add a render tier only for sites that need it).
- Non-HTML media pipelines (design the storage to allow it, don't build it).

→ Next: **[Capacity](02-systems/web-crawler/capacity.md)**
