# AI Data Assistant

A **plug-and-play Business Intelligence chatbot widget** that can be embedded into any web application with a single `<script>` tag. The system performs natural language data reasoning (NL2SQL) and connects directly to client databases — no data engineering required from the client side.

> Inspired by [REF-001 Vanna.ai](references.md#ref-001), [REF-002 Chatbase](references.md#ref-002), and [REF-003 Quivr](references.md#ref-003). See [Product References](references.md) for a full comparison.

---

## System at a Glance

```
 Client Website
 ┌──────────────────────────────────────────────────┐
 │  <script src="widget.js"                         │
 │          data-tenant-id="acme"                   │
 │          data-theme="dark">                      │
 │  </script>                      [ARC-L1-001]     │
 └────────────────┬─────────────────────────────────┘
                  │ iframe + PostMessage API
 ┌────────────────▼─────────────────────────────────┐
 │  API Gateway + Auth Layer                        │
 │  Rate Limit · CORS · JWT · Tenant Resolve        │  [ARC-L2-001..003]
 └────────────────┬─────────────────────────────────┘
                  │
 ┌────────────────▼─────────────────────────────────┐
 │  AI Core Engine                                  │
 │  ┌─────────────────┐   ┌──────────────────────┐  │
 │  │ NL2SQL Pipeline │   │ RAG Retriever        │  │
 │  │ [ARC-L3-002]    │   │ [ARC-L3-003] REF-003 │  │
 │  └────────┬────────┘   └──────────┬───────────┘  │
 │           │  Orchestrator routes  │ [ARC-L3-001]  │
 └───────────┼───────────────────────┼──────────────┘
             │                       │
 ┌───────────▼───────────────────────▼──────────────┐
 │  Data Layer                                      │
 │  BigQuery [ARC-L4-001] · Pgvector [ARC-L4-002]  │
 │  Redis Cache [ARC-L4-003]                        │
 └──────────────────────────────────────────────────┘
             │
 ┌───────────▼──────────────────────────────────────┐
 │  Infrastructure (GCP Cloud Run)     [ARC-L5-001] │
 │  Monitoring · CI/CD · Secrets                    │
 └──────────────────────────────────────────────────┘
```

---

## ID Legend

| Prefix | Scope |
|---|---|
| `ARC-L1-XXX` | Client Embed components |
| `ARC-L2-XXX` | API Gateway & Auth components |
| `ARC-L3-XXX` | AI Core Engine components |
| `ARC-L4-XXX` | Data Access components |
| `ARC-L5-XXX` | Infrastructure & Ops components |
| `TSK-XXX` | Implementation roadmap tasks |
| `DEC-XXX` | Architecture design decisions |
| `SPC-T-XXX` | Technical specifications |
| `SPC-S-XXX` | Security specifications |
| `SPC-O-XXX` | Observability specifications |
| `REF-XXX` | Product references |

---

## Quick Navigation

| Section | What's inside |
|---|---|
| [Architecture Overview](architecture/overview.md) | 5-layer system with component IDs |
| [GCP Reference](architecture/gcp-reference.md) | Cloud architecture with annotated diagram |
| [Application Services](architecture/application-services.md) | Frontend, backend, AI middleware detail |
| [Storage & Governance](architecture/storage-governance.md) | BigQuery, Dataplex, Pgvector, Redis |
| [Integration & Runtime](specs/technical.md) | Embed, serverless, DB support specs |
| [Security](specs/security.md) | JWT, SQL injection prevention, multi-tenant |
| [Observability](specs/observability.md) | Latency, token usage, cost alerts |
| [Design Decisions](planning/design-decisions.md) | 7 key architectural trade-offs with rationale |
| [Implementation Roadmap](planning/roadmap.md) | 10 tasks with inputs, outputs, criteria |
| [Product References](references.md) | Vanna.ai · Chatbase · Quivr comparisons |
