# Implementation Roadmap

Ten tasks in recommended build order. Each task lists its ID, what it requires as input, what it delivers as output, and the acceptance criteria that define "done".

```
 TSK-001 ──► TSK-002 ──► TSK-003 ──► TSK-004 ──► TSK-005
                                                      │
 TSK-010 ◄── TSK-009 ◄── TSK-008 ◄── TSK-007 ◄── TSK-006
```

---

## TSK-001 — Widget Loader JS

- [ ] **Status:** Not started

**What it is:** The `<script>` tag bootstrap. A tiny JS file (< 5 KB, minified) that reads `data-*` attributes, injects a trigger button into the host page, and lazy-loads the chat iframe on first click.

**Inputs required:** None — this is the entry point.

**Outputs delivered:**
- `widget.js` — the loader script
- Iframe spawned pointing to the Widget UI service
- PostMessage channel established between host page and iframe

**Acceptance criteria:**
- Adding `<script src="widget.js" data-tenant-id="test">` to a blank HTML page shows a chat button
- Clicking the button opens the iframe widget
- No CSS leaks from the widget into the host page
- Script weight < 5 KB gzipped

**Reference:** [REF-002 Chatbase](../references.md#ref-002) embed pattern · [ARC-L1-001](../architecture/overview.md#layer-1--client-embed)

---

## TSK-002 — API Server (FastAPI)

- [ ] **Status:** Not started

**What it is:** The core backend service. A FastAPI (Python) application deployed on Cloud Run with a health check endpoint and the skeleton of the `/v1/chat` endpoint.

**Inputs required:** TSK-001 (widget needs an endpoint to call)

**Outputs delivered:**
- `POST /v1/chat` — accepts `{message, tenant_id, session_id}`, returns `200 OK` with placeholder response
- `GET /v1/health` — returns `{"status": "ok"}` for Cloud Run health checks
- Dockerfile + Cloud Run deployment config
- GitHub Actions CI/CD deploy workflow

**Acceptance criteria:**
- `curl /v1/health` returns 200 on Cloud Run
- Cold start time < 2 seconds measured from Cloud Run metrics
- Widget can call `/v1/chat` and receive a response (even if placeholder)

**Reference:** [ARC-L2-004](../architecture/application-services.md#arc-l2-004--api-server-fastapi)

---

## TSK-003 — Auth Middleware

- [ ] **Status:** Not started

**What it is:** Tenant-key validation and JWT issuance. Every request to `/v1/chat` must carry a valid JWT. The widget exchanges the `tenant-key` (from the `data-tenant-id` attribute) for a short-lived JWT on widget init.

**Inputs required:** TSK-002 (runs as middleware on the API server)

**Outputs delivered:**
- `POST /v1/auth/token` — validates tenant-key, returns JWT (15-min TTL)
- JWT validation middleware on all `/v1/` routes
- Tenant key store (Cloud SQL table: `tenants`)

**Acceptance criteria:**
- A valid tenant-key returns a signed JWT
- An invalid key returns `401 Unauthorized`
- A request to `/v1/chat` without a JWT returns `401`
- JWT expiry is enforced — expired token returns `401`

**Reference:** [ARC-L2-002](../architecture/overview.md#layer-2--api-gateway--auth) · [SPC-S-001](../specs/security.md#spc-s-001--jwt-per-request-auth)

---

## TSK-004 — Schema Registry

- [ ] **Status:** Not started

**What it is:** The metadata store that tells the NL2SQL Engine what tables and columns exist for each tenant. Backed by Dataplex on GCP for production; a simple JSON file store for local dev.

**Inputs required:** TSK-003 (schema API must be authenticated)

**Outputs delivered:**
- `GET /v1/schema` — returns schema metadata for the authenticated tenant
- Schema loader script that introspects BigQuery and registers metadata in Dataplex
- Admin endpoint `POST /v1/schema/refresh` — triggers manual schema refresh
- Redis cache layer for schema reads (TTL 1 hour)

**Acceptance criteria:**
- Schema for a tenant returns correct table/column names after running the loader
- Schema read latency < 50ms (from Redis cache)
- `POST /v1/schema/refresh` updates the cache within 5 seconds

**Reference:** [ARC-L4-002a](../architecture/storage-governance.md#arc-l4-002a--dataplex--schema-registry) · [DEC-003](design-decisions.md#dec-003)

---

## TSK-005 — NL2SQL Pipeline

- [ ] **Status:** Not started

**What it is:** The core AI feature. Receives a natural language question, fetches schema, generates SQL via LLM, validates it, and returns the SQL ready for execution.

**Inputs required:** TSK-004 (needs schema), TSK-003 (needs tenant context)

**Outputs delivered:**
- `NL2SQLEngine` class with `generate(question, tenant_id) → sql` method
- LLM integration (Vertex AI Gemini primary)
- Few-shot prompt template with schema injection
- Auto-retry loop (up to 3 attempts on validation failure)
- Integration with Query Validator (TSK-006 dependency — stub for now)

**Acceptance criteria:**
- "Show me total sales by month" generates valid BigQuery SQL for a known schema
- Invalid SQL triggers retry; correct SQL produced within 3 attempts for > 90% of test cases
- Schema context is always included in the prompt (verified via logging)
- LLM call latency logged per request

**Reference:** [REF-001 Vanna.ai](../references.md#ref-001) · [ARC-L3-002](../architecture/application-services.md#arc-l3-002--nl2sql-pipeline)

---

## TSK-006 — BigQuery Connector + Query Validator

- [ ] **Status:** Not started

**What it is:** Two components built together. The Query Validator checks safety before execution; the BigQuery Connector executes the validated SQL and returns results.

**Inputs required:** TSK-005 (NL2SQL produces SQL to validate and execute)

**Outputs delivered:**
- `QueryValidator` class — blocks SELECT *, DDL/DML, cross-tenant refs, > 10 GB scans
- `BigQueryConnector` class — executes SQL, returns `{columns, rows, bytes_scanned}`
- Error feedback format for NL2SQL retry loop
- Query execution logged with bytes scanned and cost estimate

**Acceptance criteria:**
- `SELECT * FROM orders` is blocked by the validator
- `DROP TABLE orders` is blocked
- A valid column-specific query executes and returns correct rows
- Bytes scanned and estimated cost logged for every query

**Reference:** [ARC-L3-003](../architecture/overview.md#layer-3--ai-core-engine) · [DEC-005](design-decisions.md#dec-005)

---

## TSK-007 — Response Formatter

- [ ] **Status:** Not started

**What it is:** Converts raw query results or RAG text into structured output that the widget's Result Renderer can display.

**Inputs required:** TSK-006 (raw BigQuery results), TSK-009 (raw RAG text, partial dependency)

**Outputs delivered:**
- `ResponseFormatter` class with `format(results, question) → {type, payload}` method
- Format types: `narrative`, `table`, `chart`, `csv_link`
- LLM-based format selector (fast model classifies question type to pick format)
- Chart.js config generator for trend/comparison data

**Acceptance criteria:**
- "What were total sales last month?" → narrative format
- "Show me all orders from March" (< 50 rows) → table format
- "Show me revenue by month for the last year" → chart format (line chart)
- "Export all customer records" (> 100 rows) → CSV download link

**Reference:** [ARC-L3-004](../architecture/overview.md#layer-3--ai-core-engine)

---

## TSK-008 — Session Manager

- [ ] **Status:** Not started

**What it is:** Multi-turn conversation context. Each chat session retains the last 10 message turns so the user can ask follow-up questions ("break that down by region" after a revenue question).

**Inputs required:** TSK-002 (API server must exist), Redis Memorystore must be provisioned

**Outputs delivered:**
- `SessionManager` class — `create()`, `get(session_id)`, `append(session_id, message)`, `expire(session_id)`
- Redis connection pool in FastAPI app
- Session ID generated on widget init, passed in every `/v1/chat` request
- Session context injected into Orchestrator prompt for follow-up resolution

**Acceptance criteria:**
- Follow-up question "break that down by region" after a revenue question generates correct SQL referencing the previous query's context
- Session expires after 30 minutes of inactivity (verified by TTL)
- New session created automatically after expiry

**Reference:** [ARC-L2-005](../architecture/application-services.md#arc-l2-005--session-manager) · [ARC-L4-003](../architecture/storage-governance.md#arc-l4-003--redis-memorystore-cache)

---

## TSK-009 — RAG Retriever

- [ ] **Status:** Not started

**What it is:** The knowledge base path for non-analytical questions. Clients upload documents (PDFs, CSVs, Markdown); these are chunked, embedded, and stored in Pgvector. Questions classified as "knowledge" by the Orchestrator are answered via top-K retrieval.

**Inputs required:** TSK-003 (auth), Cloud SQL + Pgvector provisioned

**Outputs delivered:**
- Document ingestion pipeline: upload → chunk → embed → store in Pgvector
- `RAGRetriever` class — `retrieve(question, tenant_id) → [chunks]` method
- Synthesis prompt: LLM generates answer grounded in retrieved chunks with citations
- `POST /v1/documents` — endpoint to upload a document for ingestion

**Acceptance criteria:**
- Uploading a PDF and asking a question about its content returns a relevant answer with source citation
- Top-5 retrieved chunks are relevant to the question (manual evaluation on 20 test questions)
- Ingestion pipeline processes a 50-page PDF in < 60 seconds

**Reference:** [REF-003 Quivr](../references.md#ref-003) · [ARC-L3-003a](../architecture/application-services.md#arc-l3-003a--rag-retriever)

---

## TSK-010 — Observability

- [ ] **Status:** Not started

**What it is:** Structured logging, distributed tracing, and cost alerting across all services. Ensures the platform is operationally visible and cost anomalies are caught before they become expensive surprises.

**Inputs required:** All previous tasks (instruments every component)

**Outputs delivered:**
- Structured JSON logs from every service (request ID, tenant ID, latency, token count, bytes scanned)
- GCP Cloud Monitoring dashboard with key metrics
- Cost alert: Cloud Monitoring alarm when a tenant's daily BigQuery spend > threshold
- Token usage report: daily summary per tenant
- SQL error rate alert: alarm when NL2SQL retry rate > 20% over 1 hour

**Acceptance criteria:**
- Every `/v1/chat` request produces a log entry with: latency breakdown, token count, SQL generated, bytes scanned, cache hit/miss
- Cost alert fires within 5 minutes of threshold breach (tested with a large query)
- Grafana / Cloud Monitoring dashboard shows all key signals in one view

**Reference:** [ARC-L5-002](../architecture/overview.md#layer-5--infrastructure--ops) · [SPC-O-001](../specs/observability.md)
