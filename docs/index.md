# AI Data Assistant

A **plug-and-play Business Intelligence chatbot widget** that can be embedded into any web application with a single `<script>` tag. The system performs natural language data reasoning (NL2SQL) and connects directly to client databases.

**Key references:** Vanna.ai (NL2SQL), Quivr (Knowledge Base), Chatbase (Widget UX)

---

## Quick Navigation

| Section | What's inside |
|---|---|
| [Architecture Overview](architecture/overview.md) | 5-layer system design tables |
| [GCP Reference](architecture/gcp-reference.md) | Cloud architecture diagram |
| [Application Services](architecture/application-services.md) | Frontend, backend, AI middleware breakdown |
| [Storage & Governance](architecture/storage-governance.md) | BigQuery, Dataplex, Pgvector, Redis |
| [Integration & Runtime](specs/technical.md) | Single-tag embed, serverless, DB support |
| [Security](specs/security.md) | JWT, SQL injection prevention, multi-tenant |
| [Observability](specs/observability.md) | Latency, token usage, cost alerts |
| [Design Decisions](planning/design-decisions.md) | Key architectural trade-offs |
| [Implementation Roadmap](planning/roadmap.md) | 10-step build order |

---

## System at a Glance

```
Widget (Shadow DOM)
    ↓ iframe + PostMessage
API Server (Cloud Run / FastAPI)
    ↓ JWT auth + tenant resolution
Agent Orchestrator
    ├── NL2SQL → BigQuery
    └── RAG → Pgvector
    ↓
Response Formatter → SSE stream → Widget UI
```
