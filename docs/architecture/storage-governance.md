# Storage & Governance — Detail Breakdown

## BigQuery (Data Warehouse)

- **Purpose:** Main data store for all client business data — transactions, logs, events, historical reports
- **Why columnar:** Only reads columns needed by query → fast for analytics even on 100M+ row tables
- **Structure:** Project → Dataset (per domain) → Tables
- **Cost model:** Charged per data scanned, NOT per query → Query Validator must prevent `SELECT *`
- **Multi-tenant:** Each client gets separate Dataset or separate Project + Row Level Security

### BigQuery ↔ AI Agent flow

1. User asks natural language question
2. NL2SQL reads schema from Data Catalog (Dataplex)
3. AI generates SQL with schema context
4. Query Validator checks SQL safety
5. BigQuery executes in parallel across workers (columnar, only relevant columns)
6. Results returned to AI Agent
7. Response Formatter converts to narrative
8. Result cached in Redis (TTL 1hr)

---

## Dataplex — Data Catalog

- **Purpose:** Stores metadata about data (table names, column descriptions, relationships, access rules)
- **Critical for NL2SQL:** AI reads this before generating SQL to know the schema without probing DB directly
- **Also handles:** Data lineage, access control, governance policies

---

## Cloud SQL + Pgvector (Operational DB + Vector Store)

- **Operational use:** Real-time data — user sessions, tenant config, chat history
- **Vector use:** Pgvector extension stores embeddings for RAG — enables semantic search over documents
- **Dual role:** Active working data (not archive like BigQuery)

---

## Cloud Storage (Object Storage)

- **Purpose:** Files that aren't structured data — CSV uploads, PDFs, model artifacts, backups, log files
- **RAG use:** Documents uploaded by clients for knowledge base embedding

---

## Redis Memorystore (Cache)

- **Purpose:** In-memory super-fast cache
- **Stores:** Frequent query results, active session tokens, rate limit counters per tenant
- **Impact:** Same question twice → served from Redis without hitting DB or calling AI model → saves cost + reduces latency dramatically
- **TTL:** Query cache 1hr, session 30min
