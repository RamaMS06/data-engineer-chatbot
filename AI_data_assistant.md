# AI Data Assistant — Project Context & SDD
> Generated from design session. Paste this into Claude Code as project context.

---

## Project Vision

Build a **plug-and-play Business Intelligence chatbot widget** that can be embedded into any web application with a single `<script>` tag. The system performs natural language data reasoning (NL2SQL) and connects directly to client databases.

**Key references:** Vanna.ai (NL2SQL), Quivr (Knowledge Base), Chatbase (Widget UX)

---

## System Architecture — 5 Layers

### Layer 1 — Client Embed
| Component | Description | Tech |
|---|---|---|
| Widget Loader JS | Single script tag, lazy-loads iframe chatbot | Vanilla JS |
| Chat Widget UI | Shadow DOM isolated UI, framework agnostic | Vanilla JS + Shadow DOM |
| Config via Attributes | `data-tenant-id`, `data-theme`, `data-db-schema` on script tag | HTML data attributes |

### Layer 2 — API Gateway & Auth
| Component | Description | Tech |
|---|---|---|
| API Gateway | Routing, rate limiting, CORS | AWS API GW / Cloudflare Workers |
| Auth Middleware | Tenant-key validation, JWT short-lived tokens | Stateless JWT |
| Tenant Resolver | Resolve schema & DB credentials by tenant-id | Multi-tenant isolation |

### Layer 3 — AI Core Engine
| Component | Description | Tech |
|---|---|---|
| Agent Orchestrator | Manages conversation flow, tool calling, memory context, intent routing | Python |
| NL2SQL Engine | Translates natural language → SQL, schema-aware prompting, auto-retry | Vanna.ai inspired |
| Query Validator | SQL validation before execution, injection prevention, access control | Pre-execution safety |
| Response Formatter | Formats query results into narrative, table, or chart output | Post-processing |

### Layer 4 — Data Access
| Component | Description | Tech |
|---|---|---|
| DB Connector | Connects to PostgreSQL, MySQL, BigQuery, Snowflake via adapter pattern | Multi-DB |
| Schema Registry | Caches DB metadata per tenant, used by NL2SQL for context | Dataplex / custom |
| Query Cache | Caches frequent query results | Redis / DynamoDB TTL |

### Layer 5 — Infrastructure & Ops
| Component | Description | Tech |
|---|---|---|
| Serverless Runtime | Auto-sleep after 20 min idle, $0 cost when inactive | Cloud Run |
| Observability | Centralized logging, per-request tracing, cost anomaly alerts | GCP Monitoring |
| CI/CD Pipeline | Auto-deploy via GitHub Actions | GitHub Actions |
| Secret Management | DB credentials & API keys | AWS Secrets Manager / Vault |

---

## GCP Reference Architecture (from diagram)

Based on Google Cloud Platform's Data Science AI Agent reference architecture:

```
User / Other System
    ↓
Active Directory / Azure AD → Identity Provider (Cloud Identity)
    ↓
Application Services (Cloud Run)
    ├── Frontend (Widget UI, Streaming, Result Renderer)
    ├── Backend (API Server, Session Manager)
    └── AI Middleware (Orchestrator, NL2SQL, RAG Retriever)
    ↓                          ↓
AI/ML Layer              Storage & Governance
├── Vertex AI ADK        ├── BigQuery (DWH)
├── Model Garden         ├── Dataplex (Data Catalog)
├── Google ADK           ├── Cloud SQL + Pgvector
└── Grounding Search     ├── Cloud Storage (Object)
                         └── Redis Memorystore (Cache)
    ↓
Ops Layer: Monitoring, Cloud IAM, Cloud Build, Artifact Registry, Secrets Manager, Logging
```

---

## Application Services — Detail Breakdown

### Frontend Sub-layer
- **Widget Chat UI** — Vanilla JS + Shadow DOM, PostMessage API for iframe comms, SSE for streaming
- **Streaming Renderer** — Real-time token display via Server-Sent Events, typing indicator
- **Result Renderer** — Render query results as table/chart (Chart.js/D3) inside chat bubble, CSV export

### Backend Sub-layer
- **API Server (Cloud Run)** — FastAPI (Python) or Express (Node), auto-scale to zero after 20min idle, cold start target <2s
- **Session Manager** — Redis session store with 30min TTL, maintains multi-turn conversation context

### AI Middleware Sub-layer
- **Agent Orchestrator** — Core routing brain. Decides: NL2SQL path vs RAG path vs hybrid. Manages tool calling.
- **NL2SQL Pipeline** — Schema injection from registry → SQL generation → validation → BigQuery execution → auto-retry on error
- **RAG Retriever** — For non-data questions ("what is churn?"), searches Pgvector embeddings, top-K retrieval

### Request Flows

```
# NL2SQL Path (analytical questions)
Widget UI → API Server → Auth check → Orchestrator → NL2SQL → BigQuery → Formatter → Stream to UI

# RAG Path (knowledge questions)
Widget UI → API Server → Orchestrator → RAG Retriever → Pgvector → LLM Generate → Stream to UI

# Cache Hit Path
Widget UI → API Server → Redis cache → Return to UI  ⚡ <50ms
```

---

## Storage & Governance — Detail Breakdown

### BigQuery (Data Warehouse)
- **Purpose:** Main data store for all client business data — transactions, logs, events, historical reports
- **Why columnar:** Only reads columns needed by query → fast for analytics even on 100M+ row tables
- **Structure:** Project → Dataset (per domain) → Tables
- **Cost model:** Charged per data scanned, NOT per query → Query Validator must prevent `SELECT *`
- **Multi-tenant:** Each client gets separate Dataset or separate Project + Row Level Security

**BigQuery ↔ AI Agent flow:**
1. User asks natural language question
2. NL2SQL reads schema from Data Catalog (Dataplex)
3. AI generates SQL with schema context
4. Query Validator checks SQL safety
5. BigQuery executes in parallel across workers (columnar, only relevant columns)
6. Results returned to AI Agent
7. Response Formatter converts to narrative
8. Result cached in Redis (TTL 1hr)

### Dataplex — Data Catalog
- **Purpose:** Stores metadata about data (table names, column descriptions, relationships, access rules)
- **Critical for NL2SQL:** AI reads this before generating SQL to know the schema without probing DB directly
- **Also handles:** Data lineage, access control, governance policies

### Cloud SQL + Pgvector (Operational DB + Vector Store)
- **Operational use:** Real-time data — user sessions, tenant config, chat history
- **Vector use:** Pgvector extension stores embeddings for RAG — enables semantic search over documents
- **Dual role:** Active working data (not archive like BigQuery)

### Cloud Storage (Object Storage)
- **Purpose:** Files that aren't structured data — CSV uploads, PDFs, model artifacts, backups, log files
- **RAG use:** Documents uploaded by clients for knowledge base embedding

### Redis Memorystore (Cache)
- **Purpose:** In-memory super-fast cache
- **Stores:** Frequent query results, active session tokens, rate limit counters per tenant
- **Impact:** Same question twice → served from Redis without hitting DB or calling AI model → saves cost + reduces latency dramatically
- **TTL:** Query cache 1hr, session 30min

---

## Technical Specifications

### Integration
- Single `<script>` tag activation
- Config via HTML `data-*` attributes
- Shadow DOM for CSS isolation
- PostMessage API for iframe ↔ parent communication

### AI & Reasoning
- NL2SQL with schema-aware prompting
- Support for local models (Ollama) and cloud APIs (Vertex AI Gemini, Claude API)
- RAG for client knowledge base
- Query validation before execution
- Fallback chain if primary model fails

### Serverless & Cost
- Auto-sleep after 20 minutes idle
- Cold start target < 2 seconds
- Pay-per-invocation pricing model
- $0 cost when system is inactive

### Database Support
- PostgreSQL, MySQL, SQLite
- BigQuery, Snowflake, Redshift
- Adapter pattern for extensibility
- Automatic schema introspection

### Security
- JWT validation per request
- SQL injection prevention in validator
- Row-level security per tenant
- Rate limiting per tenant-key
- Secrets managed in Secrets Manager / Vault

### Observability
- Latency tracking per pipeline step
- Token usage logging per query
- SQL error rate monitoring
- Cost alert per tenant per day

### AI Model Options
- Vertex AI — Gemini (cloud, GCP native)
- Anthropic Claude API (external)
- Local model via Ollama (cost optimization)
- Fallback chain if model fails

---

## Key Design Decisions

1. **Serverless-first** — Cloud Run chosen for zero idle cost. Client pays only when widget is actively used.
2. **Shadow DOM isolation** — Widget CSS cannot bleed into host application, host CSS cannot break widget.
3. **Schema Registry as NL2SQL context** — AI never probes DB directly for schema; always reads from cached catalog. Faster + more secure.
4. **Redis in front of BigQuery** — BigQuery has 24hr built-in cache, but only for identical queries. Redis handles near-identical questions with different phrasing.
5. **Query Validator is non-negotiable** — BigQuery charges per data scanned. A bad `SELECT *` on a 1TB table = expensive. Validator prevents this.
6. **Dual RAG + NL2SQL paths** — Orchestrator routes: analytical questions → BigQuery via SQL; knowledge questions → Pgvector via embedding search.
7. **Multi-tenant via Dataset isolation** — Each client's data in separate BigQuery Dataset + Row Level Security for defense in depth.

---

## Recommended Next Steps (Implementation Order)

1. **Widget Loader JS** — Basic script tag embed with iframe + PostMessage
2. **API Server (FastAPI)** — Single endpoint, health check, Cloud Run deploy
3. **Auth Middleware** — Tenant-key validation + JWT issue
4. **Schema Registry** — Store client schema metadata, expose to NL2SQL
5. **NL2SQL Pipeline** — Core feature, schema injection + SQL generation + validation
6. **BigQuery Connector** — Execute validated SQL, return results
7. **Response Formatter** — Narrative + table/chart render in widget
8. **Session Manager** — Multi-turn conversation context via Redis
9. **RAG Retriever** — Pgvector embeddings for knowledge base questions
10. **Observability** — Logging, cost tracking, error monitoring

