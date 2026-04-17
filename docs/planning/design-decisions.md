# Key Design Decisions

1. **Serverless-first** — Cloud Run chosen for zero idle cost. Client pays only when widget is actively used.

2. **Shadow DOM isolation** — Widget CSS cannot bleed into host application, host CSS cannot break widget.

3. **Schema Registry as NL2SQL context** — AI never probes DB directly for schema; always reads from cached catalog. Faster + more secure.

4. **Redis in front of BigQuery** — BigQuery has 24hr built-in cache, but only for identical queries. Redis handles near-identical questions with different phrasing.

5. **Query Validator is non-negotiable** — BigQuery charges per data scanned. A bad `SELECT *` on a 1TB table = expensive. Validator prevents this.

6. **Dual RAG + NL2SQL paths** — Orchestrator routes: analytical questions → BigQuery via SQL; knowledge questions → Pgvector via embedding search.

7. **Multi-tenant via Dataset isolation** — Each client's data in separate BigQuery Dataset + Row Level Security for defense in depth.
