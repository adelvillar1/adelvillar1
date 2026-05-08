# 👋 Hey, I'm Alejandro Del Villar

I build **B2B SaaS products** — full-stack, solo-founder, production at scale. I ship AI that isn't a chatbot wrapper, databases that aren't just Postgres, and pipelines that don't need humans.

---

## 📐 The Scale

| What | Count | Why it matters |
|------|-------|----------------|
| Ships tracked | **687** active, with full cabin maps, dining venues, and bar inventories | Every travel advisor needs ship-level detail instantly |
| Cruise itineraries | **86,000** active (~67K with future sailings) | Matched to ships, ports, seasons, and traveler profiles |
| Port visits indexed | **630,000+** ship-port pairs | Every ship, every port, every sail date |
| Route corridors | **16,000+** unique sea routes | The atomic unit of cruise intelligence — a specific ship on a specific route in a specific season |
| Ports with deep data | **2,377** — weather, marine conditions, air quality, cost indices | Full destination intelligence, not just coordinates |
| FalkorDB graph nodes | ~**25,000** across 9 node types | Ships, ports, routes, clans, amenities, guest profiles — all connected |
| AI Chat responses | **40,000** prewarmed in PG+Redis cache | ~$0.003/query blended, instant response, no LLM call needed |
| Pipeline steps orchestrated | **47** across 10 phases | Runs without human intervention after the scraper finishes |
| LLM inference done | **1.7M** calls | …

For **$100** flat-rate (Ollama Cloud). Would have cost **~$280K** on per-token pricing. That's **2,800x cheaper** — and every insight shipped to production.

---

## ⚡ What Most People Haven't Done

### Inference Arbitrage — Not Just "Use Cheap Models"

I run a **multi-tier model routing strategy** across 4 providers. Every task gets matched to the cheapest adequate model — not averaged, not guessed, not sticking to one provider:

| Tier | Model | Provider | Cost structure |
|------|-------|----------|----------------|
| Fast chat + compression | DeepSeek V4 Flash | DeepSeek API | ~$0.15/M tokens, 1M context |
| Reasoning + subagents | Kimi K2.6 | Kimi/Moonshot | Token-based (cheap) |
| **Bulk inference** | 7+ models (GLM-5, Gemma 4:31b, Nemotron, etc.) | **Ollama Cloud flat-rate** | **~$100/mo unlimited** |

The 1.7M insight precompute that cost $100 on Ollama Cloud flat-rate would have been $280K on DeepSeek or OpenAI. Most people don't even know Ollama Cloud exists. Most people don't think in terms of pricing-model arbitrage.

### FalkorDB — A Graph Database That Speaks Redis Protocol

Instead of using a vector database, Neo4j, or traditional SQL for the knowledge graph, I use **FalkorDB** — an in-memory graph database that speaks the Redis wire protocol. This means:

- Every existing Redis client library works with it (no new drivers)
- Cypher queries for graph traversal (16K route corridors → 57 clans → 24 regions)
- Full Redis commands still work (BGSAVE, replication, TTL)
- **200MB idle** — runs comfortably on a $5/month Railway service
- Replication via **RDB binary copy**: BGSAVE → SCP the dump → RESTORE on production

The replication trick alone took 3 failed approaches before finding this — DUMP/RESTORE doesn't work with FalkorDB's `graphdata` key type, and streaming script-based replication is too slow for 25K nodes.

### Prisma Soft-Delete Middleware (That Actually Works)

Not "add `deletedAt` to every model and remember to filter." A **single `$extends` interceptor** that:

```
findMany → auto-appends WHERE deleted_at IS NULL
delete   → transparently converts to UPDATE SET deleted_at = NOW()
Models without deleted_at → whitelisted in a config set → throws before the query hits the DB
```

One-time setup. Zero leaky abstractions. Every team member forgets soft-delete exists — and that's the point.

### Dual-Emit LLM Generation

One LLM call produces **two outputs simultaneously**: structured JSON columns (enums, arrays, booleans) and natural-language narration. The model emits `{"narration": "...", "bestFitPersona": "luxury", "riskFlags": ["weather", "capacity"] ...}` — both the queryable database column and the human-facing text in a single inference. No second pass, no hallucination drift between structured and unstructured outputs.

### Scraper Worker: From 9GB to 200MB Memory

A single Playwright browser for all scraper jobs consumed **9GB+ of RAM** and OOM-killed the service weekly. The fix: **kill the browser after every port**. Spawn Playwright → scrape port → close → spawn again. Per-port lifecycle instead of singleton. Dropped memory to ~200MB. Zero OOMs since.

### PDF Extraction: Spatial Text Reordering

Standard PDF text extraction reads left-to-right, top-to-bottom. Two-column supplier invoices come back as interleaved garbage. The fix: extract every text fragment with its bounding box, **group by Y coordinate** (same row = same Y), **sort by X** within each row, then concatenate rows top-to-bottom. Runs via `pdfjs-dist` subprocess for sandbox isolation. Haiku 4.5 vision then extracts structured fields from the reordered text.

### Route Map SVGs from Raw Map Data

No Google Maps API, no Mapbox subscription, no OSM tiles. I download **Natural Earth 10m coastline vectors**, run them through a custom SVG pipeline, and generate route-map SVGs server-side at +40ms each. PG caches them. Redis serves them. Zero API cost per map. Batch prewarm script generates all 16K corridor maps on deploy.

### A 47-Step Pipeline That Manages Itself

The post-scraper pipeline has **10 phases, 47 steps, 36 unique scripts** — corridor computation, enrichment, insight generation, graph projection, production sync. Each script default to incremental (new/changed-only, `--force` for full runs). A **DB-backed orchestrator** (`lib/pipeline-jobs/auto-advance.ts`) manages the lifecycle:

```
start → execute step → advance to next → handle failures → pause/resume → cancel/restart
```

Every step shows output, duration, error. The pipeline logs JSONL events to a `pipeline_runs.log` column with auto-trim to 500 lines. A step detail panel in the admin UI lets you click any step and see exactly what happened.

**All 36 scripts were audited for schema drift** — every raw SQL column verified against `@map` decorators, every Prisma `data:` field verified against model scalars. Two bugs found. Zero additional drift.

---

## 🚀 What I'm Building

### [Cruising Intelligence](https://cruisingintelligence.com) · [`cruise-intelligence`](https://github.com/adelvillar1/cruise-intelligence)
The engine behind everything above. B2B SaaS for travel advisors. AI Chat with 40 tools, persona-based insights, route maps, FalkorDB knowledge graph, automated pipeline. Next.js · TypeScript · PostgreSQL · Prisma · FalkorDB · Redis · Ollama Cloud

### [Trip Ledger](https://thetripledger.com) · [`commissiontracker`](https://github.com/adelvillar1/commissiontracker)
Multi-agency platform for travel agencies — bookings, supplier commission reconciliation, payroll. AI vision PDF extraction with spatial text reordering. Multi-tenant row-level security. Soft-delete everywhere. Next.js · TypeScript · PostgreSQL · Prisma

### TTESS · [`TTESS`](https://github.com/adelvillar1/TTESS)
Teacher evaluation management — T-TESS (16 dimensions) and T-PESS (5 domains). Domain-chunked parallel LLM generation producing `.docx` reports with evidence tables, executive summaries, growth plans. Baseline stamp: where vision data exists, the model's inferred rating is discarded. Next.js · TypeScript · PostgreSQL · Prisma · Ollama Cloud

### [Cruiser Intelligence](https://github.com/adelvillar1/cruiserintelligence)
Consumer cruise mobile app (React Native, iOS + Android). 14-endpoint API on the CI data engine. 7 screens, quiz flow, climate conditions, upsell widgets.

### [Beacon](https://github.com/adelvillar1/beacon)
Resource Allocation & Workstream Planning Dashboard. Python/FastAPI.

### [Rider Scout](riderscout-saas)
Multi-tenant school pickup/dropoff queue management. WebSocket real-time updates, subdomain isolation, Stripe billing. FastAPI · PostgreSQL · React/Vite · Redis · Celery

---

## 🛠 The Toolchain Behind the Patterns

**Languages:** TypeScript (primary), Python (scripts, FastAPI), SQL (Postgres, FalkorDB Cypher), React Native  
**Infrastructure:** Railway (10+ services), Docker, Natural Earth (10m coastline data)  
**Database:** PostgreSQL (2M+ records), FalkorDB (25K graph nodes), Redis (40K prewarmed cache entries)  
**AI:** DeepSeek V4 Flash, Kimi K2.6, Ollama Cloud (flat-rate bulk), Haiku 4.5 (vision), Gemini 2.5  
**Testing:** Vitest, Playwright (E2E + scraper), Pytest

---

*Building in public. Solo founder. Production scale. One commit at a time.*
