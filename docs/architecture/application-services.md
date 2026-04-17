# Application Services — Detail Breakdown

## Frontend Sub-layer

- **Widget Chat UI** — Vanilla JS + Shadow DOM, PostMessage API for iframe comms, SSE for streaming
- **Streaming Renderer** — Real-time token display via Server-Sent Events, typing indicator
- **Result Renderer** — Render query results as table/chart (Chart.js/D3) inside chat bubble, CSV export

## Backend Sub-layer

- **API Server (Cloud Run)** — FastAPI (Python) or Express (Node), auto-scale to zero after 20min idle, cold start target <2s
- **Session Manager** — Redis session store with 30min TTL, maintains multi-turn conversation context

## AI Middleware Sub-layer

- **Agent Orchestrator** — Core routing brain. Decides: NL2SQL path vs RAG path vs hybrid. Manages tool calling.
- **NL2SQL Pipeline** — Schema injection from registry → SQL generation → validation → BigQuery execution → auto-retry on error
- **RAG Retriever** — For non-data questions ("what is churn?"), searches Pgvector embeddings, top-K retrieval

---

## AI Model Options

| Provider | Model | Use Case |
|---|---|---|
| Vertex AI | Gemini | Cloud, GCP native |
| Anthropic | Claude API | External, high capability |
| Ollama | Local model | Cost optimization |

Fallback chain is triggered if any model fails.
