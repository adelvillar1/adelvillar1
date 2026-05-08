# 👋 Hey, I'm Alejandro Del Villar

I build **B2B SaaS products** — full-stack, solo-founder, production at scale. I ship AI that isn't a chatbot wrapper, databases that aren't just Postgres, and pipelines that don't need humans.

---

## 📐 The Scale

| What | Count | Why it matters |
|------|-------|----------------|
| Ships tracked | **687** active | Full cabin maps, dining venues, bar inventories — advisor-ready |
| Cruise itineraries | **86,000** active (~67K with future sailings) | Matched to ships, ports, seasons, traveler profiles |
| Route corridors | **16,000+** unique sea routes | The atomic intelligence unit: a specific ship × route × season |
| Ports with deep data | **2,377** — weather, marine conditions, air quality, cost indices | Full destination intelligence, not coordinates on a map |
| Port ship visits | **630,000+** | Every ship, every port, every sail date |
| FalkorDB graph | ~**25,000** nodes (9 types) + **1,010** CABIN_FIT edges | Ships, ports, clans, amenities, guest profiles — all traversable by Cypher |
| AI Chat responses | **40,000** precomputed, cached in PG+Redis | ~$0.003/query blended — instant, no LLM call on cache hit |
| Pipeline steps | **47** across 10 phases, 36 unique scripts | Runs without human intervention after the scraper finishes |
| LLM inferences | **1.7M** corridor insights generated | For **$100** flat-rate (Ollama Cloud) — would cost **~$280K** on per-token pricing |

---

## ⚡ The AI Patterns (Most People Haven't Done This)

### 1. Seven-Tier Structured AI Pipeline (T1–T7)

Precomputing AI narration is one thing. What I built is a **tiered insight hierarchy** — seven distinct types of AI-generated intelligence, each targeting a different question travel advisors actually ask, all running as a batch pipeline after every scraper cycle:

| Tier | What it answers | How it works |
|------|----------------|--------------|
| **T1 — Best Fit** | "Which traveler types love this route?" | Per-corridor × persona × season: luxury, couples, solo, families, adventure, budget |
| **T2 — Upgrade Paths** | "Where's the value sweet spot?" | Compares price vs experience across similar corridors |
| **T3 — Capacity** | "Is this sailing likely to sell out?" | Historical crowd patterns, seasonal peaks |
| **T4 — Family Variation** | "How do sister ships differ on this route?" | Within-family corridor comparisons |
| **T5 — Risk Flags** | "What should I warn my client about?" | Hurricane windows, wave heights, port instability, cost spikes |
| **T6 — Ship × Corridor Fit** | "Which specific ship is best for this route?" | Per-ship-per-corridor joint fit analysis, the recommendation atom |
| **T7 — Line Personality** | "What's this cruise line's vibe?" | Archetype, defining traits, seasonal mode (e.g., "luxury class, Mediterranean summers, Alaska novelty") |

Every tier runs post-pipeline. Every result is precomputed — not generated at query time. The advisor clicks a badge and gets instant, persona-steered narration. **1.7M cells generated**, delivered at Redis speed.

### 2. Domain-Chunked AI Generation

Instead of throwing an entire T-TESS evaluation (16 dimensions) at an LLM in one call — which produces shallow, generic narratives — I split it into **4 concurrent domain-scoped calls** (one per T-TESS domain). Each call handles 4–5 dimensions against a focused prompt. Results merge into one coherent analysis.

Then the **baseline stamp** kicks in: wherever vision-extracted ratings exist, they **deterministically overwrite** the model's inferred rating. The discrepancy is logged: `[chunked] baseline stamp X.X: model=Y → extracted=Z`. The model's guess at a rating is literally discarded wherever real data exists.

Same pattern for T-PESS (5 domains, 23 indicators) — but that's 5 parallel calls, 3-element justification pattern per indicator, one comprehensive `.docx` report with cover page, evidence table, executive summary, artifact table, growth plan, and overall assessment.

### 3. AI Vision Extraction, Not Chat

PDF booking confirmations arrive in every possible format — one-column, two-column, tables, prose blocks, multi-page. Standard text extraction leaves all the structure on the floor. Here's the actual pipeline:

```
PDF upload → pdfjs-dist extracts raw text fragments
           → Spatial reorder: group by Y coordinate, sort by X
           → @napi-rs/canvas renders pages as images
           → Haiku 4.5 vision extracts booking number, price, commission, dates
           → Returns structured JSON → creates booking record
```

The spatial reorder is the part most people miss — two-column supplier invoices come back as interleaved garbage without it. Haiku vision alone can't fix bad text input; the reordering is what makes the extraction reliable. And it runs via `pdfjs-dist` subprocess for sandbox isolation so a bad PDF can't crash the server.

### 4. Precomputed AI Chat Responses

40,000 AI chat responses are **prewarmed before anyone asks anything**. The AI Chat doesn't generate from scratch on every query:

```
Query → Regex classifier (zero cost, instant)
      → Redis cache hit  → instant response ($0.00)
      → PG cache hit     → write-through to Redis, return ($0.00)
      → Both miss        → live DeepSeek synthesis, persist to both ($0.003)
```

Three dedicated prewarm scripts (v2 → v3 → v4, 4–5h total on Max tier) seed the cache with responses for ships, cruise lines, rankings, regions, ports, comparisons, and corridor intelligence. The blended cost lands at ~$0.003/query — **97% cheaper** than running every query through a single Sonnet call.

Every cache hit bumps `hitCount` and `lastHitAt`, enabling "regenerate top misses" jobs that improve the corpus over time without manual curation.

### 5. Bulk AI — Batch Precompute at Flat-Rate

**1.7 million** corridor insight cells. Each one is an LLM call (prompt + corridor record + persona + season). Running them through a per-token API would cost **~$280K**.

Instead, I route the entire batch job through **Ollama Cloud flat-rate ($100/month unlimited)**. The bulk tier handles 7+ different models (GLM-5, Gemma 4:31b, Nemotron 30b, MiniMax M2.7, etc.) — whatever fits the task. 1.7M inferences for $100. That's a **2,800x cost advantage** from one routing decision.

This isn't theoretical — it's shipped. Every one of those 1.7M cells is live on cruisingintelligence.com, served to travel advisors daily.

---

## 🚀 What I'm Building

### [Cruising Intelligence](https://cruisingintelligence.com) · [`cruise-intelligence`](https://github.com/adelvillar1/cruise-intelligence)
The engine behind all the AI patterns above. B2B SaaS for travel advisors. FalkorDB knowledge graph, 47-step automated pipeline, persona-based insights on every surface, route map SVGs from raw coastline data.

**Stack**: Next.js · TypeScript · PostgreSQL · Prisma · FalkorDB · Redis · Ollama Cloud · DeepSeek

### [Trip Ledger](https://thetripledger.com) · [`commissiontracker`](https://github.com/adelvillar1/commissiontracker)
Multi-agency platform for travel agencies — bookings, supplier commission reconciliation, agent payroll. AI vision PDF extraction with spatial text reordering. Multi-tenant row-level security. Soft-delete everywhere.

**Stack**: Next.js · TypeScript · PostgreSQL · Prisma · Tailwind CSS

### TTESS · [`TTESS`](https://github.com/adelvillar1/TTESS)
Teacher evaluation management — T-TESS (16 dimensions) and T-PESS (5 domains). Domain-chunked parallel LLM generation with baseline stamp. Produces `.docx` reports with evidence tables, executive summaries, and growth plans.

**Stack**: Next.js · TypeScript · PostgreSQL · Prisma · Ollama Cloud

### [Cruiser Intelligence](https://github.com/adelvillar1/cruiserintelligence)
Consumer cruise mobile app (React Native, iOS + Android). 14-endpoint API on the CI data engine. Quiz flow, climate conditions, upsell widgets.

### [Beacon](https://github.com/adelvillar1/beacon)
Resource Allocation & Workstream Planning Dashboard. Python/FastAPI.

### [Rider Scout](riderscout-saas)
Multi-tenant school pickup/dropoff queue management. WebSocket real-time updates, subdomain isolation, Stripe billing. FastAPI · PostgreSQL · React/Vite · Redis · Celery

---

## 🛠 The Infrastructure Behind the Patterns

| Layer | What | Why |
|-------|------|-----|
| **Orchestration** | DB-backed auto-advancing pipeline, 47 steps, event log (JSONL), step-level triage | Pipeline runs without babysitting |
| **Cache** | PG = source of truth, Redis = speed cache, 2-layer read-through | Redis flush doesn't destroy the corpus |
| **Graph** | FalkorDB (in-memory graph, speaks Redis protocol). 25K nodes, Cypher queries | No new drivers, no vector DB, no Neo4j bill |
| **Replication** | RDB binary copy (BGSAVE → SCP → RESTORE) | DUMP/RESTORE fails on graphdata key types |
| **Scraper** | Per-port Playwright lifecycle, 9GB → 200MB memory | Weekly OOM kills are a solved problem |
| **Cache warm** | 4 dedicated prewarm scripts, 40K+ responses seeded before first user query | Users never wait for an LLM |
| **Cost strategy** | 4-provider model routing, flat-rate bulk tier, 2,800x arbitrage advantage | $100 does what $280K would |
| **Schema integrity** | All 36 pipeline scripts audited against Prisma schema, column names verified against `@map` | Zero drift between scripts and models |

---

*Building in public. Solo founder. Production scale. One commit at a time.*
