# Implementation Roadmap

Steps in recommended build order:

- [ ] **Step 1 — Widget Loader JS** — Basic script tag embed with iframe + PostMessage
- [ ] **Step 2 — API Server (FastAPI)** — Single endpoint, health check, Cloud Run deploy
- [ ] **Step 3 — Auth Middleware** — Tenant-key validation + JWT issue
- [ ] **Step 4 — Schema Registry** — Store client schema metadata, expose to NL2SQL
- [ ] **Step 5 — NL2SQL Pipeline** — Core feature: schema injection + SQL generation + validation
- [ ] **Step 6 — BigQuery Connector** — Execute validated SQL, return results
- [ ] **Step 7 — Response Formatter** — Narrative + table/chart render in widget
- [ ] **Step 8 — Session Manager** — Multi-turn conversation context via Redis
- [ ] **Step 9 — RAG Retriever** — Pgvector embeddings for knowledge base questions
- [ ] **Step 10 — Observability** — Logging, cost tracking, error monitoring
