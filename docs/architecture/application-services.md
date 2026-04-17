# Application Services — Detail Breakdown

All application services run on **Cloud Run** ([ARC-L5-001](overview.md#layer-5--infrastructure--ops)). Three sub-layers, each independently deployable.

```
 ┌──────────────────────────────────────────────────────────────────┐
 │  CLIENT BROWSER                                                  │
 │  <script> tag ──► Widget Loader ──► iframe ──► Shadow DOM UI    │
 └───────────────────────────┬──────────────────────────────────────┘
                             │ PostMessage / SSE
 ┌───────────────────────────▼──────────────────────────────────────┐
 │  FRONTEND SUB-LAYER                           ARC-L1-XXX         │
 │  Widget Chat UI · Streaming Renderer · Result Renderer           │
 └───────────────────────────┬──────────────────────────────────────┘
                             │ HTTPS / REST
 ┌───────────────────────────▼──────────────────────────────────────┐
 │  BACKEND SUB-LAYER                            ARC-L2-XXX         │
 │  FastAPI Server · Session Manager · Auth Middleware              │
 └───────────────────────────┬──────────────────────────────────────┘
                             │ Internal service call
 ┌───────────────────────────▼──────────────────────────────────────┐
 │  AI MIDDLEWARE SUB-LAYER                      ARC-L3-XXX         │
 │  Agent Orchestrator · NL2SQL Pipeline · RAG Retriever            │
 └──────────────────────────────────────────────────────────────────┘
```

---

## Frontend Sub-layer

### ARC-L1-002a — Widget Chat UI
The visible chat interface. Implemented in Vanilla JS inside an `<iframe>` with Shadow DOM for CSS isolation. Key behaviours:
- **Trigger button** — floating button in the bottom-right corner of the host page, rendered by the Widget Loader JS before the iframe is loaded
- **Lazy load** — full iframe only initialises on first user click, keeping the initial script weight under 5 KB
- **PostMessage API** — all communication between the host page and the widget iframe goes through `window.postMessage` with origin validation
- **Theme support** — light/dark/custom via `data-theme` attribute; CSS variables injected into Shadow DOM at init
- *Reference: [REF-002 Chatbase](../references.md#ref-002) — single-script embed pattern*

### ARC-L1-002b — Streaming Renderer
Handles real-time token display as the LLM generates the answer. Uses **Server-Sent Events (SSE)** from the FastAPI backend. Displays a typing indicator while waiting for the first token. Renders tokens character-by-character into the chat bubble as they arrive — same UX pattern as ChatGPT and Claude.ai.

### ARC-L1-002c — Result Renderer
Converts the structured response payload into rich visual output inside the chat bubble:

| Output type | When used | Library |
|---|---|---|
| Narrative prose | Summary questions | Plain text |
| Markdown table | Row-level data | Marked.js |
| Line / bar chart | Trend / comparison data | Chart.js |
| CSV download link | Large result sets (> 100 rows) | Blob API |

---

## Backend Sub-layer

### ARC-L2-004 — API Server (FastAPI)
The primary backend service. Exposes REST endpoints consumed by the Widget UI:

```
POST /v1/chat          ──► main chat endpoint, returns SSE stream
GET  /v1/health        ──► Cloud Run health check
POST /v1/auth/token    ──► exchange tenant-key for JWT
GET  /v1/schema        ──► return schema metadata for a tenant
```

Built with **FastAPI (Python)** for async support and automatic OpenAPI docs. Deployed on Cloud Run with:
- Min instances: 0 (scale to zero)
- Max instances: 10 per tenant (configurable)
- Cold start target: < 2 seconds (slim Docker image, pre-loaded model client)

### ARC-L2-005 — Session Manager
Maintains multi-turn conversation context. Each conversation has a `session_id` stored in **Redis Memorystore** with a 30-minute sliding TTL. The session stores:
- Last 10 message turns (sliding window to control prompt size)
- Active tenant context (resolved credentials, schema namespace)
- User preferences (preferred output format, language)

On session expiry, the next message starts a fresh session. No conversation history is persisted to long-term storage by default (configurable per tenant for compliance).

---

## AI Middleware Sub-layer

### ARC-L3-001 — Agent Orchestrator
The routing brain. Receives: `{user_message, session_history, tenant_context}`. Decision logic:

```
 ┌─────────────────────────────────┐
 │  Incoming user message          │
 └──────────────┬──────────────────┘
                │
       ┌────────▼────────┐
       │  Intent Classify │  ◄── LLM classifier (fast, cheap model)
       └────────┬─────────┘
                │
        ┌───────┴────────┐
        │                │
   Analytical?      Knowledge?
        │                │
   NL2SQL path      RAG path
  [ARC-L3-002]     [ARC-L3-003a]
        │                │
        └───────┬────────┘
                │
      ┌─────────▼──────────┐
      │ Response Formatter  │  [ARC-L3-004]
      └─────────────────────┘
```

Supports **multi-LLM fallback**: Vertex AI Gemini (primary) → Anthropic Claude API (secondary) → Ollama local (cost fallback). If the primary model fails or times out, the next in chain is tried automatically.

### ARC-L3-002 — NL2SQL Pipeline
Step-by-step process for every analytical question:

```
 1. Fetch schema   ──► Schema Registry (Dataplex)
 2. Build prompt   ──► system prompt + schema DDL + few-shot SQL examples
 3. LLM generates  ──► Vertex AI / Claude API
 4. Validate SQL   ──► Query Validator [ARC-L3-003]
    ├── PASS ──► Execute on BigQuery [ARC-L4-001]
    └── FAIL ──► Re-prompt with error (up to 3 retries)
 5. Return results ──► Response Formatter [ARC-L3-004]
```

*Reference: [REF-001 Vanna.ai](../references.md#ref-001) — schema-aware prompting and auto-retry pattern*

### ARC-L3-003a — RAG Retriever
For knowledge questions ("what is churn?", "explain this metric"):

```
 1. Embed question  ──► text-embedding-004 (Vertex AI)
 2. Top-K search    ──► Pgvector cosine similarity (K=5 default)
 3. Retrieve chunks ──► ranked document passages
 4. Synthesize      ──► LLM generates answer grounded in retrieved text
 5. Cite sources    ──► document name + page number appended to response
```

*Reference: [REF-003 Quivr](../references.md#ref-003) — opinionated RAG defaults (chunk size, top-K, citation format)*

---

## AI Model Options

| Priority | Provider | Model | Use case |
|---|---|---|---|
| 1 (primary) | Vertex AI | Gemini 1.5 Pro | NL2SQL + RAG, GCP native, lowest latency |
| 2 (fallback) | Anthropic | Claude API | High accuracy on complex SQL, external |
| 3 (cost) | Ollama | Llama 3 / Mistral | Local model, zero API cost, slower |

Fallback is automatic — the Orchestrator retries the next provider on timeout or rate-limit error.
