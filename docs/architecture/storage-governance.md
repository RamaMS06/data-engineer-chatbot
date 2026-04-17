# Storage & Governance — Detail Breakdown

```
 ┌──────────────────────────────────────────────────────────────────┐
 │  QUERY PATH (fast)                                               │
 │                                                                  │
 │  Question ──► Redis Cache ──► HIT: return < 50ms                │
 │                  │                                               │
 │                MISS                                              │
 │                  │                                               │
 │                  ▼                                               │
 │  NL2SQL ──► Dataplex (schema) ──► BigQuery (execute)            │
 │                                       │                         │
 │  RAG ──► Pgvector (vector search) ────┤                         │
 │                                       │                         │
 │                              Results ─▼──► Formatter ──► UI     │
 └──────────────────────────────────────────────────────────────────┘

 ┌──────────────────────────────────────────────────────────────────┐
 │  INGEST PATH (background)                                        │
 │                                                                  │
 │  Client uploads PDF/CSV ──► Cloud Storage                       │
 │                                 │                               │
 │                           Chunking job                          │
 │                                 │                               │
 │                        Embedding model                          │
 │                                 │                               │
 │                         Pgvector store                          │
 └──────────────────────────────────────────────────────────────────┘
```

---

## ARC-L4-001 — BigQuery (Data Warehouse)

**Purpose:** Primary analytical data store for all client business data — transactions, events, logs, reports.

**Why BigQuery:**
- Columnar storage → only scans columns referenced in the SQL query → fast and cheap even on 100M+ row tables
- Serverless execution — no cluster to manage, no idle cost
- Built-in 24hr result cache for identical queries (complements our Redis cache)
- Row Level Security natively enforced at the table level per tenant

**Multi-tenant isolation strategy:**

```
 GCP Project (shared platform)
 ├── Dataset: tenant_acme
 │     ├── Table: orders
 │     ├── Table: customers
 │     └── Row Level Security: user = acme_service_account
 ├── Dataset: tenant_globex
 │     ├── Table: transactions
 │     └── Row Level Security: user = globex_service_account
 └── Dataset: tenant_initech
       └── ...
```

**Cost control:** The Query Validator ([ARC-L3-003]) blocks any SQL that would cause a full table scan. BigQuery charges per bytes scanned — a `SELECT *` on a 1 TB table could cost ~$5 per query. See [DEC-005](../planning/design-decisions.md#dec-005).

**BigQuery ↔ AI Agent flow:**

```
 1. User question
 2. NL2SQL reads schema from Dataplex  [ARC-L4-002]
 3. AI generates SQL
 4. Query Validator checks safety      [ARC-L3-003]
 5. BigQuery executes (columnar, parallel workers)
 6. Results → Response Formatter       [ARC-L3-004]
 7. Response → Redis cache (TTL 1hr)  [ARC-L4-003]
 8. Response → Widget UI via SSE
```

---

## ARC-L4-002a — Dataplex — Schema Registry

**Purpose:** Metadata catalog for every client's database schema. The NL2SQL Engine reads from here — never probes BigQuery directly for schema.

**Why a dedicated registry:**
- Probing BigQuery for schema on every query adds latency (200–500ms) and costs compute
- Dataplex descriptions allow business-friendly column names ("customer_lifetime_value" vs "clv_usd") — the AI uses these to generate more accurate SQL
- Centralised access control — who can see which tables is governed here, not in application code
- Data lineage tracking: know which queries read which columns across all tenants

**What is stored per tenant:**

| Metadata field | Example | Used by |
|---|---|---|
| Table name | `orders` | NL2SQL prompt injection |
| Column names + types | `order_total FLOAT64` | NL2SQL prompt injection |
| Column description | "Total value in USD including tax" | NL2SQL prompt injection |
| Sample values | `status: ["pending","shipped","cancelled"]` | Prevents hallucinated enum values |
| Foreign keys | `orders.customer_id → customers.id` | JOIN generation |

Schema is refreshed daily via a scheduled Cloud Run job, or on-demand via admin API when the client adds tables.

---

## ARC-L4-002b — Cloud SQL + Pgvector (Operational DB + Vector Store)

**Dual role — same managed instance, two purposes:**

### Operational (Cloud SQL)
Real-time working data that is not archived in BigQuery:
- User sessions (30-min TTL, also in Redis — Cloud SQL is the durable backup)
- Tenant configuration (API keys, widget settings, schema namespace mappings)
- Chat history (optional, configurable per tenant for compliance)
- Audit log (which user asked what, when)

### Vector Store (Pgvector extension)
The embedding database for RAG retrieval:
- Client documents (PDFs, CSVs, Markdown) are chunked, embedded, and stored here
- `pgvector` extension adds a `vector(1536)` column type and cosine similarity operators
- Top-K retrieval query: `ORDER BY embedding <=> $1 LIMIT 5`

*Reference: [REF-003 Quivr](../references.md#ref-003) — PGVector-native RAG architecture*

**Why one instance for both:** At startup scale, operating two separate managed databases doubles fixed cost. The operational and vector workloads have complementary access patterns (operational: high-frequency small reads; vector: low-frequency large similarity scans) — they don't compete for resources.

---

## ARC-L4-001b — Cloud Storage (Object Store)

**Purpose:** Files that are not structured data.

| Use case | Content | Lifecycle |
|---|---|---|
| RAG document uploads | PDFs, CSVs, Markdown files from clients | Kept until client deletes; triggers embedding pipeline on upload |
| Model artifacts | Fine-tuned model weights (future) | Versioned |
| Container image layers | Cached by Artifact Registry | Managed by CI/CD |
| Log archives | Exported Cloud Logging data | 90-day retention |

Object upload triggers a **Cloud Run ingestion job** that chunks the document, calls the embedding model, and writes vectors to Pgvector. The original file is retained in Cloud Storage for re-embedding if the model changes.

---

## ARC-L4-003 — Redis Memorystore (Cache)

**Purpose:** Eliminate redundant LLM calls and BigQuery scans for repeated questions.

**What is cached:**

| Key | Value | TTL | Impact |
|---|---|---|---|
| `semantic_hash(question + tenant_id)` | Formatted response payload | 1 hour | Avoids LLM call + BigQuery scan for repeat questions |
| `session:{session_id}` | Conversation history (last 10 turns) | 30 min sliding | Multi-turn context without DB reads |
| `rate_limit:{tenant_id}` | Request counter | 1 min rolling | Rate limiting without DB |
| `schema:{tenant_id}` | Schema Registry snapshot | 1 hour | Avoids Dataplex read on every query |

**Why Redis in front of BigQuery's native cache:**
BigQuery's 24hr cache only matches byte-identical queries. "Show me last month's revenue" and "What was total revenue in March?" generate different SQL — neither matches the other in BigQuery's cache. Redis is keyed by the **question's semantic hash** (not the SQL), so both questions can hit the same cached answer.

See [DEC-004](../planning/design-decisions.md#dec-004).
