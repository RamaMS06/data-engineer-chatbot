# GCP Reference Architecture

Based on Google Cloud Platform's Data Science AI Agent reference architecture.

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

## Request Flow Paths

### NL2SQL Path (analytical questions)
```
Widget UI → API Server → Auth check → Orchestrator → NL2SQL → BigQuery → Formatter → Stream to UI
```

### RAG Path (knowledge questions)
```
Widget UI → API Server → Orchestrator → RAG Retriever → Pgvector → LLM Generate → Stream to UI
```

### Cache Hit Path
```
Widget UI → API Server → Redis cache → Return to UI  ⚡ <50ms
```
