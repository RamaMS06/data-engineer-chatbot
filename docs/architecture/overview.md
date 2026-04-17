# System Architecture — 5 Layers

```
+------------------------------------------------------------------+
|  LAYER 1 — Client Embed                            [ARC-L1-XXX]  |
|  Script tag → Shadow DOM Widget → PostMessage → iframe           |
+------------------------------------------------------------------+
                              │
+------------------------------------------------------------------+
|  LAYER 2 — API Gateway & Auth                      [ARC-L2-XXX]  |
|  CORS · Rate Limit · JWT Validation · Tenant Resolve             |
+------------------------------------------------------------------+
                              │
+------------------------------------------------------------------+
|  LAYER 3 — AI Core Engine                          [ARC-L3-XXX]  |
|  Orchestrator ──► NL2SQL ──► BigQuery                            |
|              └──► RAG    ──► Pgvector                            |
+------------------------------------------------------------------+
                              │
+------------------------------------------------------------------+
|  LAYER 4 — Data Access                             [ARC-L4-XXX]  |
|  BigQuery · Dataplex Schema Registry · Redis Cache · Pgvector    |
+------------------------------------------------------------------+
                              │
+------------------------------------------------------------------+
|  LAYER 5 — Infrastructure & Ops                    [ARC-L5-XXX]  |
|  Cloud Run · GCP Monitoring · GitHub Actions · Secrets Manager   |
+------------------------------------------------------------------+
```

---

## Layer 1 — Client Embed

| ID | Component | Description | Tech | Notes |
|---|---|---|---|---|
| ARC-L1-001 | Widget Loader JS | A tiny (< 5 KB) bootstrap script delivered via CDN. Inserted with a single `<script>` tag. Lazy-loads the full chat iframe only when the user interacts with the trigger button — zero impact on host page load time. | Vanilla JS | Inspired by [REF-002 Chatbase](../references.md#ref-002) embed pattern |
| ARC-L1-002 | Chat Widget UI | The full chat interface runs inside an `<iframe>` with Shadow DOM for style isolation. The widget is framework-agnostic — works inside React, Vue, Angular, or plain HTML host apps without any dependency conflicts. | Vanilla JS + Shadow DOM | Shadow DOM prevents host CSS bleed in both directions |
| ARC-L1-003 | Config via Attributes | All client-side configuration is passed via HTML `data-*` attributes on the script tag. No JavaScript API required. Example: `data-tenant-id="acme"`, `data-theme="dark"`, `data-db-schema="sales"`. The loader reads these attributes and forwards them to the iframe on init. | HTML data attributes | Tenant ID is required; all others are optional with defaults |

---

## Layer 2 — API Gateway & Auth

| ID | Component | Description | Tech | Notes |
|---|---|---|---|---|
| ARC-L2-001 | API Gateway | The public entry point for all widget requests. Handles: CORS allowlist enforcement, per-tenant rate limiting (requests/minute), request routing to backend services, and DDoS protection. Stateless — no session stored here. | AWS API GW / Cloudflare Workers | Cloudflare Workers preferred for global edge latency |
| ARC-L2-002 | Auth Middleware | Validates the `tenant-key` included in every request header. On success, issues a short-lived JWT (15-minute TTL) scoped to the tenant. All downstream services trust the JWT — they do not re-validate against the key store. Prevents credential exposure on the client side. | Stateless JWT | Tenant key is a long-lived secret; JWT is ephemeral |
| ARC-L2-003 | Tenant Resolver | Resolves a `tenant-id` into: the correct DB connection credentials, the schema registry namespace, and the allowed query scope. Credentials are fetched from Secrets Manager at resolve time — never stored in the application layer. | Multi-tenant isolation | Each tenant's credentials are isolated — compromise of one does not expose others |

---

## Layer 3 — AI Core Engine

| ID | Component | Description | Tech | Notes |
|---|---|---|---|---|
| ARC-L3-001 | Agent Orchestrator | The central routing brain. Receives the user's message and conversation history, classifies the intent (analytical data question vs. knowledge question vs. ambiguous), and dispatches to either the NL2SQL pipeline or the RAG retriever. Also manages tool-calling loops, retry logic, and response assembly. Supports multi-turn context via session memory. | Python | Inspired by [REF-001 Vanna.ai](../references.md#ref-001) agent pattern |
| ARC-L3-002 | NL2SQL Engine | Translates a natural language question into a validated SQL query. Process: (1) fetch schema context from Schema Registry, (2) inject schema into system prompt, (3) call LLM to generate SQL, (4) pass to Query Validator, (5) if invalid, retry with error as feedback (up to 3 attempts). Uses few-shot examples of known good queries to improve accuracy. | Python + LLM API | Inspired by [REF-001 Vanna.ai](../references.md#ref-001) NL2SQL pipeline |
| ARC-L3-003 | Query Validator | Intercepts every generated SQL statement before execution. Checks: no `SELECT *` (cost control), no DDL/DML (`DROP`, `INSERT`, `UPDATE`, `DELETE`), no cross-tenant table references, SQL syntax validity, query complexity budget (estimated bytes scanned). Rejects and returns an error to the NL2SQL Engine for retry. | Pre-execution safety | Non-negotiable — see [DEC-005](../planning/design-decisions.md#dec-005) |
| ARC-L3-004 | Response Formatter | Takes raw query results (rows + columns) or RAG text and converts them into the appropriate output format: narrative prose for summary questions, Markdown table for row-level data, Chart.js config for trend/comparison questions, or a CSV download link for large exports. The format is chosen by the LLM based on the question type. | Post-processing | Rendered by the widget's Result Renderer ([ARC-L1-002]) |

---

## Layer 4 — Data Access

| ID | Component | Description | Tech | Notes |
|---|---|---|---|---|
| ARC-L4-001 | DB Connector | Adapter-pattern connector supporting multiple database backends. Each adapter implements a common interface: `connect()`, `execute(sql)`, `introspect_schema()`. Connectors exist for PostgreSQL, MySQL, SQLite (dev), BigQuery, Snowflake, and Redshift. New connectors can be added without touching the NL2SQL engine. | Multi-DB adapter | BigQuery is the primary production target; others via adapter |
| ARC-L4-002 | Schema Registry | A metadata cache per tenant. Stores: table names, column names + types, descriptions, foreign key relationships, and sample values for enum columns. The NL2SQL Engine always reads from here — never probes the live DB for schema. Schema is refreshed on a schedule (daily) or on-demand via admin API. Backed by Dataplex on GCP. | Dataplex / custom cache | See [DEC-003](../planning/design-decisions.md#dec-003) — AI never touches live DB for schema |
| ARC-L4-003 | Query Cache | Redis-backed result cache keyed by a semantic hash of the question + tenant ID. Before any LLM call, the Orchestrator checks the cache. Cache hit → return in < 50 ms. Cache miss → full pipeline. Results are cached with a 1-hour TTL. Distinct from BigQuery's native 24hr cache, which only matches byte-identical queries. | Redis / DynamoDB TTL | See [DEC-004](../planning/design-decisions.md#dec-004) — handles paraphrased repeat questions |

---

## Layer 5 — Infrastructure & Ops

| ID | Component | Description | Tech | Notes |
|---|---|---|---|---|
| ARC-L5-001 | Serverless Runtime | All backend services run as Cloud Run containers. Auto-scale from zero to N instances based on request load. Auto-sleep after 20 minutes of inactivity — $0 cost when idle. Cold start target < 2 seconds (achieved via container pre-warming and slim image). | Cloud Run | See [DEC-001](../planning/design-decisions.md#dec-001) |
| ARC-L5-002 | Observability | Centralized structured logging via GCP Cloud Logging. Distributed tracing per request with latency breakdown per pipeline step. Cost anomaly alerts when a tenant's daily BigQuery spend exceeds threshold. Token usage logged per query for LLM cost attribution. | GCP Monitoring + Cloud Logging | See [docs/specs/observability.md](../specs/observability.md) |
| ARC-L5-003 | CI/CD Pipeline | GitHub Actions workflow triggers on every push to `main`. Runs: lint → unit tests → Docker build → push to Artifact Registry → deploy to Cloud Run. Zero-downtime rolling deploy. Rollback via Cloud Run revision traffic splitting. | GitHub Actions | `.github/workflows/deploy.yml` in repo |
| ARC-L5-004 | Secret Management | All credentials (DB passwords, API keys, tenant secrets) stored in GCP Secret Manager or HashiCorp Vault. Secrets are injected at runtime via environment variables — never baked into container images or source code. Rotation is automated for DB credentials. | Secret Manager / Vault | Tenant DB credentials fetched per-request by [ARC-L2-003] |
