# GCP Reference Architecture

Based on Google Cloud Platform's Data Science AI Agent reference architecture, adapted for multi-tenant widget deployment.

```
 ┌─────────────────────────────────────────────────────────────────────┐
 │  IDENTITY & ACCESS                                                  │
 │  Azure AD / Active Directory ──► Cloud Identity (GCP IAM)          │
 └─────────────────────────────────┬───────────────────────────────────┘
                                   │ Authenticated user context
 ┌─────────────────────────────────▼───────────────────────────────────┐
 │  APPLICATION SERVICES  (Cloud Run — auto-scale, zero idle cost)     │
 │                                                                     │
 │  ┌─────────────────┐  ┌──────────────────┐  ┌───────────────────┐  │
 │  │ Frontend        │  │ Backend          │  │ AI Middleware     │  │
 │  │ Widget UI       │  │ FastAPI Server   │  │ Orchestrator      │  │
 │  │ SSE Renderer    │  │ Session Manager  │  │ NL2SQL Pipeline   │  │
 │  │ Result Renderer │  │ Auth Middleware  │  │ RAG Retriever     │  │
 │  └────────┬────────┘  └───────┬──────────┘  └────────┬──────────┘  │
 └───────────┼───────────────────┼─────────────────────┼─────────────┘
             │                   │                     │
 ┌───────────▼───────────────────▼──────────────────── ▼─────────────┐
 │  AI / ML LAYER                     STORAGE & GOVERNANCE            │
 │                                                                     │
 │  ┌────────────────────┐     ┌───────────────────────────────────┐  │
 │  │ Vertex AI ADK      │     │ BigQuery        (DWH)  ARC-L4-001 │  │
 │  │ Gemini (cloud LLM) │     │ Dataplex   (Schema Reg) ARC-L4-002│  │
 │  │ Model Garden       │     │ Cloud SQL + Pgvector    ARC-L4-002 │  │
 │  │ Grounding Search   │     │ Cloud Storage  (Objects)          │  │
 │  │ Ollama (local LLM) │     │ Redis Memorystore (Cache) ARC-L4-003│ │
 │  └────────────────────┘     └───────────────────────────────────┘  │
 └─────────────────────────────────────────────────────────────────────┘
             │
 ┌───────────▼─────────────────────────────────────────────────────────┐
 │  OPS LAYER                                              ARC-L5-XXX  │
 │  Cloud Monitoring · Cloud IAM · Cloud Build · Artifact Registry     │
 │  Secret Manager · Cloud Logging · GitHub Actions CI/CD              │
 └─────────────────────────────────────────────────────────────────────┘
```

---

## GCP Service Roles

### Cloud Run (ARC-L5-001)
Hosts all three application sub-layers (frontend, backend, AI middleware) as separate containerised services. Scales to zero after 20 minutes idle — no compute cost when the widget is not in use. Each service has its own auto-scaling policy: the AI Middleware scales more aggressively (LLM calls are slow) while the frontend scales conservatively (mostly static).

### Vertex AI + Model Garden
The primary LLM provider. Gemini models are used for NL2SQL generation and RAG synthesis. Model Garden provides access to other foundational models without leaving GCP's security boundary. Grounding Search enables web-augmented answers for general questions outside the client's data.

### BigQuery (ARC-L4-001)
The analytical data warehouse. Client business data lives here. Columnar storage means only queried columns are scanned — critical for cost control. Each tenant has a dedicated Dataset with Row Level Security applied at the table level.

### Dataplex — Schema Registry (ARC-L4-002)
GCP's managed data catalog. Stores schema metadata (table descriptions, column types, business glossary terms) that the NL2SQL Engine reads before generating any SQL. Dataplex also enforces data governance policies and tracks data lineage across pipelines.

### Cloud SQL + Pgvector (ARC-L4-002)
Dual-purpose operational database. As Cloud SQL: stores user sessions, tenant configuration, chat history. As Pgvector: hosts the vector embeddings for RAG retrieval. Using a single managed instance for both roles simplifies operations and reduces cost at startup scale.

### Redis Memorystore (ARC-L4-003)
In-memory cache layer. Query results cached by semantic hash for 1 hour. Session tokens stored with 30-minute TTL. Rate limit counters per tenant stored here. At peak load, the majority of repeat questions are served directly from Redis without touching BigQuery or the LLM.

### Cloud Storage
Object store for unstructured data: PDF/CSV document uploads from clients (ingested into Pgvector for RAG), model artifacts, container image layers, and log archives. Not in the real-time query path.

---

## Request Flows

### NL2SQL Path — analytical questions
```
Widget UI
  │  user asks "what were total sales last month?"
  ▼
API Server  ──► Auth check (JWT) ──► Tenant Resolve
  │
  ▼
Orchestrator  ──► classifies as: analytical → NL2SQL
  │
  ▼
NL2SQL Engine
  ├── fetch schema from Dataplex
  ├── inject schema into LLM prompt
  ├── Vertex AI generates SQL
  ├── Query Validator checks safety
  └── BigQuery executes query
  │
  ▼
Response Formatter  ──► narrative + table
  │
  ▼
Stream to Widget UI via SSE            ⚡ target < 3s
```

### RAG Path — knowledge questions
```
Widget UI
  │  user asks "what does churn rate mean?"
  ▼
Orchestrator  ──► classifies as: knowledge → RAG
  │
  ▼
RAG Retriever
  ├── embed question → vector
  ├── top-K search in Pgvector
  └── retrieve relevant document chunks
  │
  ▼
Vertex AI / Claude  ──► synthesize answer from chunks
  │
  ▼
Stream to Widget UI via SSE
```

### Cache Hit Path
```
Widget UI
  │  user asks a previously answered question
  ▼
Orchestrator  ──► semantic hash lookup in Redis
  │
  └──► Cache HIT  ──► return cached result    ⚡ < 50ms
  └──► Cache MISS ──► full NL2SQL or RAG path
```
