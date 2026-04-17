# System Architecture — 5 Layers

## Layer 1 — Client Embed

| Component | Description | Tech |
|---|---|---|
| Widget Loader JS | Single script tag, lazy-loads iframe chatbot | Vanilla JS |
| Chat Widget UI | Shadow DOM isolated UI, framework agnostic | Vanilla JS + Shadow DOM |
| Config via Attributes | `data-tenant-id`, `data-theme`, `data-db-schema` on script tag | HTML data attributes |

## Layer 2 — API Gateway & Auth

| Component | Description | Tech |
|---|---|---|
| API Gateway | Routing, rate limiting, CORS | AWS API GW / Cloudflare Workers |
| Auth Middleware | Tenant-key validation, JWT short-lived tokens | Stateless JWT |
| Tenant Resolver | Resolve schema & DB credentials by tenant-id | Multi-tenant isolation |

## Layer 3 — AI Core Engine

| Component | Description | Tech |
|---|---|---|
| Agent Orchestrator | Manages conversation flow, tool calling, memory context, intent routing | Python |
| NL2SQL Engine | Translates natural language → SQL, schema-aware prompting, auto-retry | Vanna.ai inspired |
| Query Validator | SQL validation before execution, injection prevention, access control | Pre-execution safety |
| Response Formatter | Formats query results into narrative, table, or chart output | Post-processing |

## Layer 4 — Data Access

| Component | Description | Tech |
|---|---|---|
| DB Connector | Connects to PostgreSQL, MySQL, BigQuery, Snowflake via adapter pattern | Multi-DB |
| Schema Registry | Caches DB metadata per tenant, used by NL2SQL for context | Dataplex / custom |
| Query Cache | Caches frequent query results | Redis / DynamoDB TTL |

## Layer 5 — Infrastructure & Ops

| Component | Description | Tech |
|---|---|---|
| Serverless Runtime | Auto-sleep after 20 min idle, $0 cost when inactive | Cloud Run |
| Observability | Centralized logging, per-request tracing, cost anomaly alerts | GCP Monitoring |
| CI/CD Pipeline | Auto-deploy via GitHub Actions | GitHub Actions |
| Secret Management | DB credentials & API keys | AWS Secrets Manager / Vault |
