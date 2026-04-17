# AI Data Assistant — Implementation Playbook
> **Source of truth for all AI code generation.**
> Path: `docs/implementation-book.md` — drop this into Claude Code at session start.
> Contains: architecture decisions, build order, plugin stack, and coding conventions.

---

## 🏷️ Tags Reference
Use these tags to navigate or grep sections:

| Tag | Meaning |
|---|---|
| `#ARCH` | Architecture decision |
| `#SPEC` | Technical specification |
| `#BUILD` | Implementation task / build step |
| `#PLUGIN` | Claude Code plugin or MCP server |
| `#CONV` | Coding convention — always follow |
| `#WARN` | Known risk or gotcha |
| `#TODO` | Not yet decided / needs discussion |

---

## Project Overview `#ARCH`

**Product:** Plug-and-play BI chatbot widget embedded via single `<script>` tag.
**Core capability:** Natural language → SQL → answer (NL2SQL), plus RAG for knowledge questions.
**Pricing model:** Serverless, $0 cost when idle. Clients pay per active usage.

**Inspired by:**
- Vanna.ai → NL2SQL approach
- Quivr → Knowledge base / RAG
- Chatbase → Widget UX and embed mechanism

---

## Stack Decisions `#ARCH` `#SPEC`

| Layer | Technology | Reason |
|---|---|---|
| Widget embed | Vanilla JS + Shadow DOM | Zero dependencies, works in React/Vue/plain HTML |
| API server | FastAPI (Python 3.12) | Async support, Pydantic validation, fast cold start |
| Serverless runtime | Google Cloud Run | Auto-scale to zero, 20min idle shutdown |
| AI orchestration | Custom Python agent | Full control over tool calling and routing logic |
| NL2SQL | Schema-aware prompting + Vanna pattern | Proven approach, schema injection = accurate SQL |
| Primary DB | BigQuery | Columnar, scalable, pay-per-scan analytics |
| Operational DB | Cloud SQL (PostgreSQL 15) | Sessions, config, chat history |
| Vector store | Pgvector (on Cloud SQL) | RAG embeddings, same DB = simpler infra |
| Cache | Redis (Cloud Memorystore) | Query cache, session store, rate limiting |
| Object storage | Google Cloud Storage | CSV uploads, PDFs, model artifacts |
| Auth | JWT (short-lived) + tenant-key | Stateless, multi-tenant safe |
| CI/CD | GitHub Actions → Cloud Run | Standard, well-documented |
| Secrets | Google Secret Manager | Never in env vars or code |
| Schema catalog | Dataplex (or custom registry) | NL2SQL context source, not DB probing |
| AI models | Vertex AI Gemini (primary), Claude API (fallback), Ollama (local dev) | Cost optimization per environment |

---

## System Architecture — 5 Layers `#ARCH`

```
┌─────────────────────────────────────────────────────┐
│  LAYER 1 — CLIENT EMBED                             │
│  Widget Loader JS │ Chat Widget UI │ Config Attrs   │
└────────────────────────┬────────────────────────────┘
                         │ HTTPS + SSE
┌────────────────────────▼────────────────────────────┐
│  LAYER 2 — API GATEWAY & AUTH                       │
│  API Gateway │ Auth Middleware │ Tenant Resolver    │
└────────────────────────┬────────────────────────────┘
                         │ validated request
┌────────────────────────▼────────────────────────────┐
│  LAYER 3 — AI CORE ENGINE                           │
│  Agent Orchestrator                                 │
│    ├── NL2SQL Pipeline                              │
│    ├── RAG Retriever                                │
│    └── Response Formatter                           │
└──────────┬─────────────────────────┬────────────────┘
           │ SQL query               │ vector search
┌──────────▼──────────┐   ┌─────────▼──────────────┐
│  LAYER 4 — DATA     │   │  LAYER 4 — DATA        │
│  BigQuery (DWH)     │   │  Pgvector (embeddings) │
│  Schema Registry    │   │  Redis (cache)         │
│  DB Connector       │   │  Cloud Storage (files) │
└─────────────────────┘   └────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  LAYER 5 — INFRA & OPS                              │
│  Cloud Run │ Monitoring │ CI/CD │ Secret Manager   │
└─────────────────────────────────────────────────────┘
```

---

## Request Flow `#ARCH`

### NL2SQL path (analytical questions)
```
User question
  → Widget UI (SSE stream open)
  → API Server (auth + rate limit check)
  → Agent Orchestrator (intent: data query)
  → Schema Registry (fetch table metadata)
  → NL2SQL Engine (inject schema + generate SQL)
  → Query Validator (safety check)
  → BigQuery (execute, return rows)
  → Response Formatter (rows → narrative + chart)
  → Stream back to Widget UI
  → Save result to Redis cache (TTL 1hr)
```

### RAG path (knowledge questions)
```
User question
  → Widget UI
  → API Server
  → Agent Orchestrator (intent: knowledge)
  → RAG Retriever (embed question → search Pgvector)
  → Top-K documents retrieved
  → LLM generates answer with document context
  → Stream back to Widget UI
```

### Cache hit path
```
User question
  → API Server
  → Redis cache hit
  → Return immediately  ⚡ target <50ms
```

---

## Build Order `#BUILD`

Work through these phases in sequence. Each phase has clear inputs and outputs.

### Phase 0 — Foundation `#BUILD`
- [ ] `0.1` Init repo structure (see Folder Structure below)
- [ ] `0.2` Setup `CLAUDE.md` in repo root
- [ ] `0.3` Setup `.mcp.json` with BigQuery + PostgreSQL MCP servers
- [ ] `0.4` Setup Google Cloud project, enable APIs (BigQuery, Cloud Run, Secret Manager)
- [ ] `0.5` Setup GitHub Actions skeleton (lint → test → deploy to Cloud Run)
- [ ] `0.6` Install Claude Code plugins (MemClaw, security-sweep, backend-architect)

**Output:** Empty but wired repo. Cloud Run deploys a hello-world. CI passes.

---

### Phase 1 — API Server & Auth `#BUILD`
- [ ] `1.1` FastAPI app skeleton with health check endpoint
- [ ] `1.2` Tenant-key validation middleware
- [ ] `1.3` JWT issue + verify logic
- [ ] `1.4` Tenant Resolver — maps tenant-id to DB credentials from Secret Manager
- [ ] `1.5` Rate limiter per tenant-key (Redis counter)
- [ ] `1.6` Cloud Run deploy + environment variable wiring
- [ ] `1.7` Write tests for auth middleware

**Output:** `POST /chat` endpoint that authenticates requests. Rejects bad keys.

---

### Phase 2 — Schema Registry `#BUILD`
- [ ] `2.1` Schema Registry data model (tenant_id, table_name, columns, descriptions)
- [ ] `2.2` Schema ingestion script — reads BigQuery dataset metadata via MCP Toolbox
- [ ] `2.3` Schema cache layer (Redis, TTL 24hr)
- [ ] `2.4` Schema Registry API — `GET /schema/{tenant_id}` for NL2SQL to consume
- [ ] `2.5` Write tests for schema fetch + cache behavior

**Output:** NL2SQL can call Schema Registry to get table context before generating SQL.

---

### Phase 3 — NL2SQL Pipeline `#BUILD`
- [ ] `3.1` Agent Orchestrator skeleton — intent classifier (SQL vs RAG vs hybrid)
- [ ] `3.2` NL2SQL Engine — schema injection into prompt template
- [ ] `3.3` SQL generation with LLM (Gemini primary, Claude fallback)
- [ ] `3.4` Query Validator — block `SELECT *`, injection patterns, unauthorized tables
- [ ] `3.5` BigQuery Connector — execute validated SQL, return results
- [ ] `3.6` Auto-retry on SQL error (re-prompt with error message, max 2 retries)
- [ ] `3.7` Response Formatter — rows → narrative string
- [ ] `3.8` Redis query cache — hash(tenant_id + question) as key, TTL 1hr
- [ ] `3.9` Write tests for validator, retry logic, and cache behavior

**Output:** `POST /chat` with a data question returns a correct SQL-derived answer.

---

### Phase 4 — Streaming & Widget `#BUILD`
- [ ] `4.1` SSE (Server-Sent Events) streaming endpoint on FastAPI
- [ ] `4.2` Widget Loader JS — single `<script>` tag, lazy iframe inject
- [ ] `4.3` Chat Widget UI — Shadow DOM, message bubbles, typing indicator
- [ ] `4.4` PostMessage bridge between iframe and parent window
- [ ] `4.5` Streaming token renderer in widget (append tokens as they arrive)
- [ ] `4.6` Result Renderer — detect table/chart in response, render with Chart.js
- [ ] `4.7` Config via data attributes (`data-tenant-id`, `data-theme`, `data-lang`)
- [ ] `4.8` Test embed in plain HTML, React app, and Vue app

**Output:** Working chatbot widget embeddable with one line. Answers stream in real-time.

---

### Phase 5 — RAG & Knowledge Base `#BUILD`
- [ ] `5.1` Embedding pipeline — PDF/text → chunks → embed → store in Pgvector
- [ ] `5.2` Document upload endpoint — `POST /knowledge/{tenant_id}/upload`
- [ ] `5.3` RAG Retriever — embed question, cosine similarity search, top-5 chunks
- [ ] `5.4` Orchestrator routing — add RAG branch to intent classifier
- [ ] `5.5` Hybrid path — combine SQL result + RAG context in single answer
- [ ] `5.6` Write tests for embedding pipeline and retrieval accuracy

**Output:** Widget can answer both "show me revenue" (SQL) and "what is our refund policy" (RAG).

---

### Phase 6 — Session & Multi-turn `#BUILD`
- [ ] `6.1` Session Manager — Redis store with conversation history per session
- [ ] `6.2` Session TTL — 30 minutes idle, configurable per tenant
- [ ] `6.3` Inject last N messages as context into Orchestrator prompt
- [ ] `6.4` Handle follow-up questions ("compare with last month" without re-stating month)
- [ ] `6.5` Session cleanup job — purge expired sessions

**Output:** Multi-turn conversations work correctly. AI remembers context within a session.

---

### Phase 7 — Observability & Hardening `#BUILD`
- [ ] `7.1` Structured logging per request (tenant_id, latency, token_count, sql_error)
- [ ] `7.2` Cost tracking per tenant per day (BigQuery bytes scanned + LLM tokens)
- [ ] `7.3` Cost alert — Cloud Monitoring alert if tenant exceeds daily threshold
- [ ] `7.4` Security sweep with `security-sweep` plugin, fix all findings
- [ ] `7.5` Load test — simulate 100 concurrent widget sessions
- [ ] `7.6` Cold start optimization — target <2s from idle
- [ ] `7.7` Final documentation with `documentation-generator` plugin

**Output:** Production-ready. Monitored, secured, documented.

---

## Folder Structure `#CONV`

```
/
├── CLAUDE.md                    ← Claude Code reads this every session
├── .mcp.json                    ← MCP server config (BigQuery, Postgres, Redis)
├── .github/workflows/           ← CI/CD pipelines
│
├── api/                         ← FastAPI application
│   ├── main.py                  ← App entrypoint
│   ├── routes/                  ← Endpoint definitions
│   │   ├── chat.py              ← POST /chat, SSE stream
│   │   ├── knowledge.py         ← POST /knowledge/upload
│   │   └── health.py            ← GET /health
│   ├── middleware/
│   │   ├── auth.py              ← Tenant-key + JWT validation
│   │   ├── rate_limit.py        ← Redis-backed per-tenant limiting
│   │   └── tenant.py            ← Tenant resolver
│   └── models/                  ← Pydantic request/response models
│
├── agents/                      ← AI core engine
│   ├── orchestrator.py          ← Intent routing (SQL vs RAG vs hybrid)
│   ├── nl2sql/
│   │   ├── engine.py            ← Schema injection + SQL generation
│   │   ├── validator.py         ← Pre-execution SQL safety check
│   │   └── prompts.py           ← Prompt templates
│   ├── rag/
│   │   ├── retriever.py         ← Pgvector similarity search
│   │   └── embedder.py          ← Text → embedding pipeline
│   └── formatter.py             ← Query results → narrative/chart hint
│
├── connectors/                  ← Database adapters
│   ├── bigquery.py              ← BigQuery execute + schema fetch
│   ├── postgres.py              ← Cloud SQL operations
│   ├── redis.py                 ← Cache + session + rate limit ops
│   └── schema_registry.py       ← Schema fetch + cache logic
│
├── widget/                      ← Frontend embed
│   ├── loader.js                ← Single script tag entry point
│   ├── widget.js                ← Shadow DOM chat UI
│   ├── renderer.js              ← Table + chart rendering
│   └── bridge.js                ← PostMessage iframe ↔ parent
│
├── scripts/                     ← Utility scripts
│   ├── ingest_schema.py         ← Populate Schema Registry from BigQuery
│   └── embed_documents.py       ← Upload docs to Pgvector
│
└── tests/
    ├── test_auth.py
    ├── test_nl2sql.py
    ├── test_validator.py
    ├── test_rag.py
    └── test_cache.py
```

---

## Coding Conventions `#CONV`

### General
- Python 3.12, strict type hints everywhere
- Pydantic v2 for all request/response models
- Async/await throughout — no blocking I/O in FastAPI routes
- All secrets via `google.cloud.secretmanager`, never in env vars or `.env` files

### SQL safety `#WARN`
- **NEVER** pass raw user input directly to any SQL string
- All SQL must pass through `agents/nl2sql/validator.py` before execution
- Block patterns: `SELECT *`, `DROP`, `DELETE`, `INSERT`, `UPDATE`, `--`, `;`
- Whitelist only: tables in Schema Registry for that tenant

### Multi-tenancy `#CONV`
- Every DB query must include `tenant_id` scope
- Redis keys always prefixed: `{tenant_id}:{key_type}:{identifier}`
- BigQuery queries scoped to tenant's Dataset only
- Log `tenant_id` in every log line

### BigQuery cost control `#WARN`
- Always use `LIMIT` clause — default 1000 rows max
- Always filter with `WHERE` on date partition columns when available
- Log bytes_processed per query for cost tracking
- Alert if single query exceeds 10GB scan

### Streaming
- All chat responses use SSE (`text/event-stream`)
- Token chunks sent as: `data: {"token": "...", "done": false}\n\n`
- Final chunk: `data: {"token": "", "done": true, "metadata": {...}}\n\n`

### Error handling
- Never expose internal error details to widget client
- Always return structured error: `{"error": "human-readable", "code": "ERR_CODE"}`
- SQL generation failure → graceful message, not stack trace

---

## Claude Code Plugin Stack `#PLUGIN`

### MCP Servers — add to `.mcp.json`

```json
{
  "mcpServers": {
    "bigquery": {
      "command": "npx",
      "args": ["-y", "@toolbox-sdk/server", "--prebuilt=bigquery"],
      "env": { "BIGQUERY_PROJECT": "YOUR_PROJECT_ID" }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@toolbox-sdk/server", "--prebuilt=postgres"],
      "env": { "POSTGRES_CONNECTION_STRING": "YOUR_CLOUD_SQL_URL" }
    },
    "redis": {
      "command": "npx",
      "args": ["-y", "redis-mcp"],
      "env": { "REDIS_URL": "YOUR_REDIS_URL" }
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@github/github-mcp-server"],
      "env": { "GITHUB_TOKEN": "YOUR_TOKEN" }
    }
  }
}
```

### Skills & Plugins — install order

```bash
# Phase 0 — install before writing any code
/plugin install memclaw@memclaw          # persistent project memory
/plugin install backend-architect        # API + DB design patterns
/plugin install security-sweep           # OWASP + LLM Top 10 scanner

# Phase 3 — when building AI core
/plugin install test-writer-fixer        # auto-generate pytest tests
/plugin install mcp-builder              # if custom MCP needed for Dataplex

# Phase 7 — final hardening
/plugin install documentation-generator  # API docs + README
/plugin install maestro-orchestrate      # multi-agent for parallel integration
```

### Repomix — for large context sessions
```bash
# Pack entire repo into single AI-readable file
npx repomix --output repomix-output.txt

# Then in Claude Code:
# "Here is the full codebase: [paste repomix-output.txt]"
```

---

## Key Architecture Decisions Log `#ARCH`

| Decision | Choice | Reason | Alternative rejected |
|---|---|---|---|
| Widget isolation | Shadow DOM | CSS cannot bleed in/out | iframe-only: too heavy, postMessage overhead |
| NL2SQL context | Schema Registry (cached) | No live DB probing = faster + safer | Probe DB directly: slow, security risk |
| Cache strategy | Redis in front of BigQuery | BQ cache only works for identical queries | BQ cache only: misses paraphrased questions |
| Multi-tenant isolation | Dataset-level separation + RLS | Defense in depth | Schema-level only: single point of failure |
| SQL execution | Validated + whitelist-only | BigQuery charges per scan; bad query = $ | Trust LLM output: injection + cost risk |
| Streaming | SSE over WebSocket | Simpler, HTTP-native, works through CDN | WebSocket: overkill for one-way token stream |
| Model fallback | Gemini → Claude → local | Cost optimize: GCP-native first | Single model: no resilience |
| Auth | JWT (15min) + refresh | Stateless, scales well | Session cookies: stateful, harder to scale |

---

## Known Risks & Gotchas `#WARN`

1. **BigQuery cold scan cost** — A `SELECT *` on a 1TB table = expensive. Query Validator is non-negotiable.

2. **Cloud Run cold start** — First request after 20min idle can be 2-3s. Mitigate with min-instances=1 for paid tiers, or warm-ping every 15min for free tier.

3. **NL2SQL hallucination** — LLM may generate plausible but wrong SQL. Always show user the SQL that was run (transparency). Add "explain mode" toggle.

4. **Pgvector performance** — Cosine similarity on 1M+ vectors gets slow without HNSW index. Set up index from day one, not after.

5. **Multi-tenant Redis key collision** — Always prefix keys with `{tenant_id}:`. A missing prefix exposes one tenant's cache to another.

6. **Shadow DOM + third-party scripts** — Some host pages run Content Security Policy that blocks inline scripts. Widget must handle CSP gracefully and surface a clear error.

7. **SSE and load balancers** — Some reverse proxies buffer SSE. Set `X-Accel-Buffering: no` header on streaming responses.

8. **Schema Registry staleness** — If client changes their DB schema, Registry goes stale. Implement webhook or polling to detect schema changes.

---

## Environment Variables `#SPEC`

```bash
# Auth
JWT_SECRET=                    # from Secret Manager
JWT_EXPIRY_MINUTES=15

# BigQuery
BIGQUERY_PROJECT_ID=
BIGQUERY_LOCATION=asia-southeast1

# Cloud SQL
CLOUD_SQL_CONNECTION_NAME=
POSTGRES_DB=
POSTGRES_USER=                 # from Secret Manager
POSTGRES_PASSWORD=             # from Secret Manager

# Redis
REDIS_URL=                     # from Secret Manager

# AI Models
VERTEX_PROJECT_ID=
VERTEX_LOCATION=asia-southeast1
ANTHROPIC_API_KEY=             # from Secret Manager (fallback)

# App config
MAX_QUERY_ROWS=1000
MAX_BIGQUERY_SCAN_GB=10
CACHE_TTL_QUERY=3600           # 1 hour
CACHE_TTL_SESSION=1800         # 30 minutes
CACHE_TTL_SCHEMA=86400         # 24 hours
IDLE_SHUTDOWN_MINUTES=20
```

---

## TODO — Open Decisions `#TODO`

- [ ] Pricing model per tenant — flat monthly vs pay-per-query?
- [ ] Widget theming system — how many variables exposed to client?
- [ ] Multi-language support — Bahasa Indonesia + English from day one?
- [ ] Dataplex vs custom Schema Registry — depends on GCP commitment level
- [ ] Ollama local model for dev — which model? (Llama 3.1, Mistral, CodeLlama?)
- [ ] Data ingestion layer — Cloud Functions for ETL from client systems?
- [ ] Admin dashboard for tenants — self-serve schema upload + API key management?

---

*Last updated: April 2026 — generated from design session context.*
