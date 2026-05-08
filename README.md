# 👋 Hey, I'm Alejandro Del Villar

I build **B2B SaaS products** — full-stack, solo-founder, production at scale. I ship AI that isn't a chatbot wrapper, databases that aren't just Postgres, and pipelines that don't need humans.

---

## 📐 The Simple Numbers

| What | Count |
|------|-------|
| Ships tracked with full cabin maps, dining, bars | **687** active |
| Cruise itineraries indexed | **86,000** active (~67K with future sailings) |
| Route corridors (the atomic intelligence unit) | **16,000+** |
| Ports with weather, marine, air quality, cost data | **2,377** |
| Port ship visits logged | **>630,000** |
| FalkorDB graph nodes (9 types, all traversable by Cypher) | ~**25,000** |
| Precomputed AI Chat responses, served at Redis speed | **40,000** (~$0.003/query blended) |
| Pipeline steps orchestrated | **47** across 10 phases, 36 unique scripts |
| LLM inferences from one batch precompute run | **1.7M** — for **$100 flat-rate** that would cost **$280K** per-token |

---

## ⚡ The Patterns You Probably Haven't Seen

### 1. Seven-Tier Structured AI Pipeline (T1–T7)

Not \"one AI model generates everything.\" Seven distinct insight types, each answering a different question a travel advisor actually asks, all precomputed post-pipeline:

| Tier | Question it answers |
|------|-------------------|
| **T1 — Best Fit** | Which traveler profile loves this route? |
| **T2 — Upgrade Paths** | Where's the hidden value? |
| **T3 — Capacity** | Will this sailing sell out? |
| **T4 — Family Variation** | How do sister ships differ here? |
| **T5 — Risk Flags** | What should I warn my client about? |
| **T6 — Ship × Corridor Fit** | Which specific ship for this route? |
| **T7 — Line Personality** | What's this cruise line's vibe? |

Every tier batch-generates after each scraper cycle. Every cell is precomputed — not generated at query time. 1.7M cells sitting in PostgreSQL, served at Redis speed, costing $100 flat-rate because the entire batch ran on Ollama Cloud instead of per-token APIs.

### 2. One LLM Call, Two Outputs (Dual-Emit)

Most AI pipelines either produce structured JSON *or* natural language narration. This does **both simultaneously** from a single inference call:

```json
{
  "narration": "This corridor peaks for families in August...",
  "fit_verdict": "strong_match",
  "risk_categories": ["weather_risk", "crowding"],
  "confidence": "high"
}
```

The model emits one JSON object where `narration` carries the prose and the remaining fields (enums, arrays, verdicts) become queryable database columns — all from the same context window, all internally consistent. If the JSON parse fails (≤5% rate), it falls back to narration-only and logs the parse failure. Every prompt is versioned so re-propagation is precise.

### 3. Four-Tier Cache Matching (Not Just a Cache)

40,000 prewarmed responses. But the lookup isn't a hash map:

```
Query → Tier 1: Entity-based key (zero cost, <1ms)
         "Tell me about Symphony of the Seas" → "ship:symphony-of-the-seas:overview"
      → Tier 2: Embedding cosine similarity (OpenAI text-embedding-3-small, 1536d)
         "Info on that big Royal Caribbean ship with water parks" → cosine 0.94 match
         In-memory Float32Array, ~245 MB for all 40K
      → Tier 3: Haiku semantic dedup ($0.001)
         Top-3 embedding candidates → "does any answer this query?" → structured JSON verdict
      → Tier 4: Live DeepSeek synthesis ($0.005)
```

Tier 2 catches paraphrase variance that a hash key misses. Tier 3 catches the gap where cosine similarity is high but the answer doesn't match. Blended cost: **~$0.001/query**.

Every precomputed response is data-grounded — the warmup scripts iterate actual PostgreSQL records (ships, corridors, ports) and generate responses from live data, not from thin air. 40K responses seeded over 4–5 hours, then cache hits serve them instantly forever.

### 4. Domain-Chunked AI with Baseline Stamp

A T-TESS evaluation has 16 dimensions. Throwing them all at an LLM in one call produces shallow, generic narratives. The fix: **4 concurrent domain-scoped calls**, one per T-TESS domain. Each produces 4–5 focused dimensions, then results merge into one coherent analysis.

Then the **baseline stamp** fires: wherever vision-extracted ratings exist, they deterministically overwrite the model's inferred ratings. The discrepancy is logged: `[chunked] baseline stamp 3.5: model=4.0 → extracted=3.5`. The model's guess at a rating is literally discarded wherever real data exists.

Same pattern for T-PESS: 5 concurrent domain calls, 23 individual indicators, 3-element justification per indicator, one `.docx` report with cover page, evidence table, executive summary, growth plan, and overall assessment.

### 5. A Graph Database That Speaks Redis

Instead of Neo4j or a vector database, the knowledge graph runs on **FalkorDB** — an in-memory graph database that speaks the Redis wire protocol. Every existing Redis client library works. Cypher queries traverse 25K nodes. Full Redis commands still work (BGSAVE, TTL, replication).

The replication trick alone took three failed approaches. DUMP/RESTORE doesn't work with FalkorDB's `graphdata` key type. The solution: **BGSAVE → SCP the RDB dump → upload → SHUTDOWN NOSAVE → redeploy**. A 243-line script (`falkordb-copy-to-production.ts`) that replaces a broken predecessor. Runs on staging, copies to production. 7 steps, `--dry-run` support, 94% confidence for a clean run.

### 6. PDF Extraction with Spatial Text Reordering

Standard PDF text extraction reads left-to-right, top-to-bottom. Two-column supplier invoices come back as interleaved garbage. The pipeline:

```
PDF → pdfjs-dist subprocess (sandbox isolation)
    → Extract every text fragment with its bounding box
    → Group by Y coordinate (same row = same Y)
    → Sort by X within each row
    → Concatenate rows top-to-bottom
    → @napi-rs/canvas renders pages as images
    → Haiku 4.5 vision extracts structured fields
    → Returns booking number, price, commission, dates as JSON
```

The spatial reorder is what makes it work — Haiku vision alone can't fix bad text input. And it runs via subprocess so a malformed PDF can't crash the server.

### 7. Scraper Worker: 9GB → 200MB by Killing the Browser

A singleton Playwright browser for all scraper jobs consumed **9GB+ of RAM** and OOM-killed the service weekly. The fix: **kill the browser after every port**.

```
Spawn Playwright → scrape port page → close browser → spawn again
```

Per-port lifecycle instead of singleton. Dropped memory to ~200MB. Zero OOMs since. The `AUTO_RESTART_HOURS=2` config exists specifically for Playwright memory-leak mitigation.

Plus: batch upserts (352 PG round-trips → 1–2 per port, ~175× DB overhead reduction), resume-from-crash (`lastScrapedBefore` timestamp), and three-way stale-port branching (force vs. freshness vs. explicit cutoff).

### 8. A 47-Step Pipeline That Manages Itself

The post-scraper cascade: corridor computation, enrichment, insight generation, graph projection, production sync. Each script defaults to incremental (new/changed-only). A **DB-backed orchestrator** (`lib/pipeline-jobs/auto-advance.ts`) manages the lifecycle:

```
start → execute step → advance to next → on failure: mark pipeline failed
                                    → on pause: finish current, stop
                                    → on resume: trigger next pending
                                    → on cancel: SIGTERM all children
                                    → on restart: create fresh run from same position
```

Every step shows output, duration, and error in the admin UI. A JSONL event log (`pipeline_runs.log`) records every state transition, auto-trimmed to 500 lines. Click any step → see its full stdout/stderr in a dark monospace panel.

**All 36 scripts were audited for schema drift** — every raw SQL column verified against `@map` decorators, every Prisma `data:` field verified against model scalars. Two bugs found. Zero additional drift.

### 9. Derived Intelligence: Finding Value in Existing Data

Most of what you need to answer interesting questions is already in the database — it just needs aggregation, cross-referencing, and a queryable shape. The pattern:

| Dimension | Source | What it gives |
|-----------|--------|---------------|
| Identity | `ports`, `ships`, `cruise_lines` | What things are |
| Behavior | `port_ship_visits`, `ship_itineraries` | When/where things happen |
| Demographics | `cruise_line_demographics` | Who the passengers are |
| Environment | `port_weekly_weather/marine/air_quality` | Conditions at a place/time |
| Economics | `country_cost_index` | Cost of being somewhere |

Every derived table uses natural keys, composite unique constraints for idempotent backfill, and stride patterns for staged rollout (80% value, 20% compute time). Results are consumed by three products from the same Postgres instance — no duplication, no sync jobs.

### 10. Pipeline Instrumentation Down to the Row

Every scraper task records structured results to `scraper_pipeline_results`: `shipsFound`, `itinerariesAdded`, `cabinsAdded`, `deploymentsDerived`, `durationMs`, `errorMessage`. A UUID `pipelineId` chains every task result to its orchestrator run.

The health endpoint has a blind spot — trigger-based scraping runs entirely outside the monthly pipeline system and shows `status: "never_run"` even when actively writing to the database. The fix: check `port_ship_visits` rows created today instead of trusting `/health`.

### Hidden Details That Matter

| Description | The trick |
|-------------|----------|
| **Prisma soft-delete** | One `$extends` interceptor transparently converts all queries. Zero leaky abstractions. No team member ever writes `WHERE deleted_at IS NULL`. |
| **Route map SVGs** | Natural Earth 10m coastline vectors, server-side SVG pipeline. **Zero API cost per map.** Batch prewarm generates all 16K on deploy. |
| **Rate limiter** | Tumbling 5-hour + sliding weekly Ollama window. Projects whether a batch fits **before starting**. |
| **Telemetry logger** | Shared `projection-telemetry.ts` across 5 FalkorDB scripts — banners, timers, ETA, batch progress, failure tracking, consistent CLI output. |
| **Cross-project KG** | Local FalkorDB indexes all project artifacts. TF-IDF + Cypher CONTAINS. No LLM embeddings, no API costs. |
| **Differential sync v3** | Zero-downtime DB sync with per-table timestamp strategy. Full upsert or skip-existing. |
| **Resume-from-crash** | Every long-running script supports `--resume` with checkpointing. Precompute, scraper, and FalkorDB projection — all idempotent. |

---

## 🧠 How I Keep It All in My Head: 154 Reusable Skills

Every non-trivial pattern I discover gets saved as a **Hermes skill** — a structured markdown file with frontmatter, triggers, numbered steps, pitfalls, and references. When I start a session, the relevant skills auto-load alongside my project context. I never re-derive a pattern I've already solved.

**154 skills** across 26 categories. The most heavily authored:

| Category | Skills | What they cover |
|----------|--------|-----------------|
| **software-development** | 51 | Dual-emit LLM, cache matching, Prisma soft-delete, pipeline orchestration, route-map SVGs, project methodology, batch UX, skill authoring |
| **creative** | 26 | ASCII video, brand asset extraction, design review, hyperframes, pixel art, slide decks, music generation |
| **devops** | 18 | Pipeline orchestration, inference arbitrage, cross-env data comparison, FalkorDB projection, scraper instrumentation, knowledge graph, postgres migrations |
| **mlops** | 14 | Model serving (vLLM), fine-tuning (Axolotl, TRL, Unsloth), evaluation, audio generation, image segmentation, DSPy |
| **github** | 6 | PR workflow, code review, issues, repo management, auth, codebase inspection |
| **autonomous-ai-agents** | 4 | Claude Code, Codex, OpenCode delegation, Hermes Agent configuration |
| **research** | 5 | arXiv, blog monitoring, Polymarket, LLM wiki, research writing |

### The patterns that recur most

| Pattern | Used in | Why it's repeatable |
|---------|---------|---------------------|
| **Dual-emit LLM** | CI (all 1.7M insights), TTESS (T-PESS) | One call produces structured columns + narration simultaneously |
| **Domain-chunked AI** | TTESS (T-TESS 16 dims), CI (T-PESS projections) | Split when 8+ fields, concurrent scoped calls, baseline stamp |
| **Multi-tier cache** | CI AI Chat (40K responses) | Entity keys → embeddings (1536d) → Haiku dedup → live synthesis |
| **Pipeline orchestrator** | CI (47 steps), any sequential job chain | DB-backed state machine, pause/resume/cancel/restart |
| **Prisma soft-delete** | Trip Ledger, TTESS, CI | One `$extends` interceptor — zero leaky abstractions |
| **Inference arbitrage** | All projects globally | 4 providers, flat-rate bulk, per-task routing |
| **Project knowledge graph** | All projects (6,528 indexed chunks) | Local FalkorDB, TF-IDF ranking, no LLM API costs |
| **Slim project methodology** | All 8 projects | warmup → plan → build → recap → wrapup cycle |

Each skill includes versioning, trigger conditions, and a pitfalls section. The project knowledge graph indexes all of them across every project — I can query "how did we do X before?" and get the matching skill, relevant recaps, and architecture docs in one result set.

---

## 🚀 What I'm Building

### [Cruising Intelligence](https://cruisingintelligence.com) · [`cruise-intelligence`](https://github.com/adelvillar1/cruise-intelligence)
The engine behind every pattern above. B2B SaaS for travel advisors. AI Chat, knowledge graph, persona insights, route maps, 47-step pipeline. Next.js · TypeScript · PostgreSQL · Prisma · FalkorDB · Redis · Ollama Cloud · DeepSeek

### [Trip Ledger](https://thetripledger.com) · [`commissiontracker`](https://github.com/adelvillar1/commissiontracker)
Multi-agency platform for travel agencies — bookings, supplier commission reconciliation, agent payroll. AI vision PDF extraction with spatial text reordering. Multi-tenant. Soft-delete everywhere. Next.js · TypeScript · PostgreSQL · Prisma

### TTESS · [`TTESS`](https://github.com/adelvillar1/TTESS)
Teacher evaluation management — T-TESS (16 dimensions) and T-PESS (5 domains). Domain-chunked parallel LLM generation with baseline stamp. `.docx` reports with evidence tables, executive summaries, growth plans. Next.js · TypeScript · PostgreSQL · Prisma · Ollama Cloud

### [Cruiser Intelligence](https://github.com/adelvillar1/cruiserintelligence)
Consumer cruise mobile app (React Native, iOS + Android). 14-endpoint API on the CI data engine. Quiz flow, climate conditions, upsell widgets.

### [Beacon](https://github.com/adelvillar1/beacon)
Resource Allocation & Workstream Planning Dashboard. Python/FastAPI.

### [Rider Scout](riderscout-saas)
Multi-tenant school pickup/dropoff queue management. WebSocket real-time updates, subdomain isolation, Stripe billing. FastAPI · PostgreSQL · React/Vite · Redis · Celery

---

*Building in public. Solo founder. Production scale. One commit at a time.*
