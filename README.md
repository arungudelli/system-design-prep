# 🧭 System Design Prep

Personal, incrementally-built prep for **FAANG Staff / Principal / EM** system-design interviews.
The goal is not to memorize architectures — it's to *derive* them from requirements, recognize reusable
patterns, reason about trade-offs, and defend decisions like a Staff/Principal Engineer.

## 📖 Read it as a website

👉 **[arungudelli.github.io/system-design-prep](https://arungudelli.github.io/system-design-prep)**

Built with **[MkDocs Material](https://squidfunk.github.io/mkdocs-material/)** — tabbed navigation, full-text
search, dark mode, and per-page tables of contents. The site is generated from the Markdown in [`docs/`](docs/)
and deployed automatically by GitHub Actions on every push to `main`.

## Repository layout

```
docs/
  index.md              # home
  00-framework/         # the 20-step method, capacity math, staff-level thinking, wording, checklist
  01-patterns/          # caching, sharding, queues & workers, idempotency, rate limiting, …
  02-systems/           # per-system deep dives (each: requirements → … → cheatsheet + principal-deep-dive)
mkdocs.yml              # site config + navigation
requirements.txt       # docs build dependencies
prompt.txt             # the source-of-truth learning plan / methodology
```

Every system is worked through the **same 20-step framework** and presented as a **staircase of "stops"** —
start with the smallest defensible design, then climb one lever at a time, always naming the trigger that
forces the next stop. Each system also ships a **★ Principal Deep Dive** (the max-scale variant + a
reconciliation table of when each heavy component is justified vs. overkill) — so you're never blind in an
interview, and never over-engineer by reflex.

## Build locally

```bash
python -m pip install -r requirements.txt
python -m mkdocs serve          # preview at http://127.0.0.1:8000
python -m mkdocs build --strict # production build (fails on broken links)
```

---

<sub>Built with Claude Code · a living document — notes get sharper after every mock.</sub>
