# 👋 Hey, I'm Alejandro Del Villar

I build **B2B SaaS products** — full-stack, solo-founder, production at scale. I'm the founder behind a portfolio of live products across travel tech, education, and logistics.

---

## 🚀 What I'm Building

### [Cruising Intelligence](https://cruisingintelligence.com) · [`cruise-intelligence`](https://github.com/adelvillar1/cruise-intelligence)
**AI-powered travel intelligence for cruise advisors.** 49 cruise lines, 687 ships, 86K itineraries — indexed and interrogable.

- **AI Chat** — 40-tool tri-stage pipeline (regex → Haiku → DeepSeek) at ~$0.003/query with PG+Redis caching and 40K pre-warmed responses
- **Persona-based insights** — per-corridor AI narration across 6 traveler profiles (couples, luxury, families, solo, etc.) surfaced on every ship, cruise line, itinerary, and region page
- **Knowledge graph** — FalkorDB with 16K RouteCorridor nodes, 2,377 ports, 57 destination clans, clan-ship cabin fit edges
- **Automated pipeline** — 47-step post-scraper cascade with full lifecycle orchestration (auto-advance, pause/resume/cancel/restart), event logging, and step-level triage
- **Route map SVGs** — server-side generated from Natural Earth 10m coastlines with batch prewarm and admin pipeline job
- **Stack**: Next.js · TypeScript · PostgreSQL · Prisma · FalkorDB · Redis · Ollama Cloud

### [Trip Ledger](https://thetripledger.com) · [`commissiontracker`](https://github.com/adelvillar1/commissiontracker)
**Multi-agency platform for travel agencies** — bookings, supplier commission reconciliation, agent payroll, sales reporting.

- **AI vision extraction** — PDF booking confirmations parsed via Haiku 4.5 vision, `@napi-rs/canvas` rendering, `pdfjs-dist` subprocess with spatial text reordering for two-column supplier invoices
- **Commission check upload** — partial submission with field-level validation highlighting
- **Multi-tenant** — row-level agency isolation via Prisma middleware
- **Soft-delete everywhere** — Prisma `$extends` middleware intercepting all queries, zero hard-deletes in the codebase
- **Stack**: Next.js · TypeScript · PostgreSQL · Prisma · Tailwind CSS

### [TTESS](https://ttess-app-production.up.railway.app) · [`TTESS`](https://github.com/adelvillar1/TTESS)
**Teacher evaluation management** — T-TESS (16 dimensions) and T-PESS (5 domains) with AI-generated analysis reports.

- **Domain-chunked LLM generation** — parallel AI calls per evaluation dimension, producing structured `.docx` reports with evidence tables, executive summaries, and growth plans
- **Multi-document upload** — PDF/DOCX/XLSX pipeline with classification and SLO-only review sessions
- **AI model: Kimi K2.6** — flat-rate Ollama Cloud, proven best for structured evaluation output
- **Stack**: Next.js · TypeScript · PostgreSQL · Prisma · Ollama Cloud

### [Cruiser Intelligence](https://github.com/adelvillar1/cruiserintelligence)
**Consumer cruise mobile app** — helping cruise passengers plan, compare, and book their perfect trip (React Native, iOS + Android).

- 14-endpoint API reading from the Cruising Intelligence data engine
- 7 screens, 4-tab navigation, quiz flow, climate conditions, upsell widgets
- **Stack**: React Native · Express · PostgreSQL

### [Beacon](https://github.com/adelvillar1/beacon)
**Resource Allocation & Workstream Planning Dashboard** — Python/FastAPI backend, Railway deployment.

### [Rider Scout](riderscout-saas)
**Multi-tenant school pickup/dropoff queue management** with real-time WebSocket updates, subdomain isolation, and Stripe billing.

- **Stack**: FastAPI · PostgreSQL · React/Vite · Redis · Celery · Stripe

---

## ⚡ How I Work

### AI-Centric Development
Every product ships serious AI — not chatbots, but production-quality extraction, analysis, and narrative generation. My inference routing strategy ("inference arbitrage") saves 90%+ on LLM costs by matching each task to the cheapest adequate model:

| Tier | Model | Use Case | Cost |
|------|-------|----------|------|
| Fast chat | DeepSeek V4 Flash | Primary chat, compression | ~$0.15/M tokens |
| Reasoning | Kimi K2.6 | Subagents, vision, curation | Flat-rate (Ollama Cloud) |
| Bulk | GLM-5, Gemma 4, Nemotron | Batch precompute, backfills | Flat-rate ~$100/mo |

### Full Methodology
Every project follows a disciplined cycle: warmup → plan → build → recap → wrapup. Plans are contracts with acceptance criteria. Recaps are journal entries with open follow-ups. A local FalkorDB knowledge graph indexes everything across all projects — cross-project recall without LLM embeddings.

### Key Patterns (79+ Custom Skills)
- **Prisma soft-delete middleware** — `$extends` interceptors, zero hard-deletes
- **Dual-emit LLM generation** — structured JSON fields + natural-language narration from one call
- **Inference arbitrage** — multi-tier model routing, flat-rate bulk providers
- **Pipeline orchestration** — DB-backed auto-advancing state machines with event logging
- **Domain-chunked AI generation** — parallel LLM calls per evaluation dimension
- **Multi-tier cache matching** — progressive cache layers for LLM responses

---

## 📊 Snapshot

| Metric | Value |
|--------|-------|
| Active projects | 8 (5 revenue-generating) |
| Custom skills discovered | 79+ |
| Production services | 10+ (Railway) |
| AI models configured | 7+ across 4 providers |
| Database records indexed | 2M+ across PostgreSQL + FalkorDB |
| Daily pipeline steps | 47 (fully orchestrated) |

---

## 🛠 Toolchain

**Languages:** TypeScript, Python, SQL, React Native  
**Backend:** Next.js, FastAPI, Express, tRPC  
**Database:** PostgreSQL, Prisma, FalkorDB (graph), Redis (cache)  
**AI:** DeepSeek, Kimi K2.6, Ollama Cloud, Haiku 4.5, Gemini 2.5  
**DevOps:** Railway, Docker, GitHub Actions  
**Testing:** Vitest, Playwright, Pytest

---

*Building in public, one commit at a time.*
