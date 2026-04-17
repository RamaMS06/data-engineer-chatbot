# Product References

Three reference products that directly informed this architecture. We borrow selectively from each — this page documents what we took, what we improved, and where we diverge.

---

## REF-001 — Vanna.ai

**Product:** Open-source NL2SQL framework — "Chat with your SQL database"
**URL:** [github.com/vanna-ai/vanna](https://github.com/vanna-ai/vanna)
**Stars:** 13,000+ GitHub stars (2025)

### What Vanna.ai does

```
User Question
     │
     ▼
Schema Context (from training data / embeddings)
     │
     ▼
LLM generates SQL  ◄── Auto-retry on error
     │
     ▼
Execute SQL on DB
     │
     ▼
Return result + chart
```

- Uses **RAG over schema DDL + example SQL pairs** to ground NL2SQL — the AI has seen your tables before generating queries
- Ships a `<vanna-chat>` web component for drop-in embed (Vanna 2.0)
- Agent-based API with user context flowing through every tool call
- Supports enterprise row-level security per user identity

### What we borrow from REF-001

| Feature | How we use it |
|---|---|
| Schema-aware prompting | `ARC-L4-002` Schema Registry feeds table/column metadata into every prompt before SQL generation |
| Auto-retry on SQL error | `ARC-L3-002` NL2SQL Engine re-prompts with the error message for up to 3 attempts |
| Agentic tool-calling pattern | `ARC-L3-001` Orchestrator uses the same tool-calling loop pattern |

### Where we differ

- Vanna is a library (embedded in your app). We are a **hosted SaaS widget** — clients don't install anything.
- Vanna focuses on a single database connection. We support **multi-tenant** with per-client schema isolation.
- We add a **Query Validator** (`DEC-005`) before execution — Vanna trusts the LLM output directly.

---

## REF-002 — Chatbase

**Product:** AI chatbot widget platform — embed on any website in seconds
**URL:** [chatbase.co](https://www.chatbase.co)

### What Chatbase does

```
Dashboard (no-code)
     │  configure knowledge base + persona
     ▼
Chatbase Platform
     │  generates embed code
     ▼
<script> tag → bubble widget or iframe
     │
     ▼
GPT-4 answers from your content
```

- Copy-paste `<script>` or `<iframe>` embed — zero engineering on the client side
- Bubble widget or full-page iframe layout options
- Connects to WhatsApp, Slack, Zendesk via integrations
- AI Actions: bot can call external APIs, not just answer questions

### What we borrow from REF-002

| Feature | How we use it |
|---|---|
| Single `<script>` tag activation | `ARC-L1-001` Widget Loader JS — identical pattern, one tag installs everything |
| Iframe isolation | `ARC-L1-002` Chat Widget UI runs in an iframe for full DOM isolation |
| Shadow DOM CSS isolation | `ARC-L1-002` Shadow DOM prevents host app styles breaking the widget |
| `data-*` config attributes | `ARC-L1-003` `data-tenant-id`, `data-theme`, `data-db-schema` on the script tag |

### Where we differ

- Chatbase is a **general knowledge chatbot** (documents, FAQs). We are a **data analytics chatbot** — we query live databases via SQL.
- Chatbase has no SQL engine. We have a full NL2SQL pipeline (`ARC-L3-002`) with BigQuery execution.
- We are **multi-tenant by design** — one platform instance serves many clients with isolated data.

---

## REF-003 — Quivr

**Product:** Open-source opinionated RAG framework — "Second Brain for your apps"
**URL:** [github.com/QuivrHQ/quivr](https://github.com/QuivrHQ/quivr)
**Stars:** 39,000+ GitHub stars (2025)

### What Quivr does

```
Document Upload (PDF, CSV, TXT, MD)
     │
     ▼
Chunking + Embedding
     │
     ▼
PGVector Store
     │
User Question ──► Top-K Retrieval ──► LLM Generate ──► Answer
```

- Opinionated RAG — ships with best-practice chunking, retrieval, and prompt templates so you don't tune from scratch
- Pluggable LLMs: OpenAI, Anthropic Claude, Ollama (local), Mistral, Groq
- PGVector-native (PostgreSQL) for vector storage
- Multi-user knowledge bases with access control

### What we borrow from REF-003

| Feature | How we use it |
|---|---|
| PGVector for embeddings | `ARC-L4-002` Cloud SQL + Pgvector extension for RAG retrieval |
| Multi-LLM fallback chain | `ARC-L3-001` Orchestrator: Vertex AI Gemini → Claude API → Ollama |
| Document ingestion pipeline | Future `TSK-009` RAG Retriever ingests client PDF/CSV uploads into Pgvector |
| Opinionated RAG defaults | We adopt Quivr's chunk size and retrieval top-K defaults as our starting point |

### Where we differ

- Quivr is a standalone knowledge base app. Our RAG is **one path inside a larger orchestrator** — the other path is NL2SQL.
- We add a **router** (`ARC-L3-001`) that decides per question: SQL or RAG. Quivr always does RAG.
- Our vector store lives in **Cloud SQL on GCP** — tightly integrated with the rest of the GCP stack.

---

## Comparison Matrix

| Dimension | REF-001 Vanna.ai | REF-002 Chatbase | REF-003 Quivr | **Our System** |
|---|---|---|---|---|
| Primary capability | NL2SQL | Knowledge chatbot | RAG framework | NL2SQL + RAG |
| Embed model | Library | Hosted widget | Self-hosted | Hosted widget |
| Multi-tenant | No | Yes (SaaS) | No | Yes |
| Live DB queries | Yes | No | No | Yes |
| Knowledge base | Via SQL history | Documents | Documents | Both |
| Deployment | Your infra | Chatbase cloud | Your infra | GCP Cloud Run |
| Pricing model | Open source | Per-message SaaS | Open source | Per-invocation serverless |
