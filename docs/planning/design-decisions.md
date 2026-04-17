# Key Design Decisions

Each decision is recorded with its ID, the context that forced the choice, alternatives that were considered, and the rationale for the final call.

---

## DEC-001 — Serverless-first (Cloud Run)

**Decision:** All backend services run on Cloud Run with scale-to-zero enabled.

**Context:** The widget is embedded on client websites with unpredictable traffic patterns — a client may have zero usage for days then spike during a marketing campaign. Traditional always-on VMs or Kubernetes clusters charge for idle capacity.

**Alternatives considered:**

| Option | Rejected because |
|---|---|
| GKE (Kubernetes) | Minimum node cost ~$150/month even at zero load |
| Cloud Functions | 9-minute timeout is insufficient for long LLM chains |
| VM (Compute Engine) | Fixed cost, manual scaling, operational overhead |

**Rationale:** Cloud Run matches cost exactly to usage. Auto-sleep after 20 minutes idle = $0 when no widget is active. Cold start < 2 seconds is acceptable for first-message latency on an infrequently-used widget.

**Trade-off:** Cold starts on first request after idle period. Mitigated by container pre-warming (Cloud Run min-instances = 1 for high-tier tenants).

---

## DEC-002 — Shadow DOM Isolation for Widget UI

**Decision:** The chat widget UI runs inside a Shadow DOM attached to an `<iframe>`, not injected directly into the host page's DOM.

**Context:** The widget is embedded in arbitrary third-party web applications with widely varying CSS frameworks (Bootstrap, Tailwind, custom CSS). Without isolation, the host app's styles break the widget, or the widget's styles leak into the host app.

**Alternatives considered:**

| Option | Rejected because |
|---|---|
| Direct DOM injection | Host CSS conflicts are undebuggable and unpredictable |
| CSS-in-JS with scoped classnames | Still vulnerable to global resets and `!important` rules |
| Custom Elements without Shadow DOM | Partial isolation only |

**Rationale:** Shadow DOM + iframe provides the strongest isolation boundary available in the browser. The widget looks and behaves identically regardless of the host application. *Adopted directly from [REF-002 Chatbase](../references.md#ref-002).*

**Trade-off:** Cross-origin iframe communication requires PostMessage API — slightly more complex than direct DOM access, but well understood and secure.

---

## DEC-003 — Schema Registry as NL2SQL Context Source

**Decision:** The NL2SQL Engine always reads schema metadata from the Schema Registry (Dataplex cache), never by probing the live database directly.

**Context:** To generate accurate SQL, the LLM needs table names, column names, types, and descriptions. The naive approach is to run `INFORMATION_SCHEMA` queries against BigQuery on each request.

**Alternatives considered:**

| Option | Rejected because |
|---|---|
| Live INFORMATION_SCHEMA query per request | +200–500ms latency, uses BigQuery compute quota |
| Hardcode schema in prompts | Breaks every time the client adds a table or column |
| Ask client to provide schema via API | Operational burden on the client |

**Rationale:** Dataplex stores metadata with richer context than `INFORMATION_SCHEMA` — column descriptions in business language, foreign key hints, sample enum values. The AI generates better SQL from business descriptions ("customer lifetime value in USD") than raw column names ("clv_usd"). Schema refreshes daily with a manual override. *Aligns with [REF-001 Vanna.ai](../references.md#ref-001) training data approach.*

**Trade-off:** Schema cache can be stale up to 24 hours. Mitigated by on-demand refresh via admin API.

---

## DEC-004 — Redis Cache in Front of BigQuery

**Decision:** A Redis semantic cache sits in front of all BigQuery queries, keyed by a hash of the question + tenant ID.

**Context:** The same business question gets asked repeatedly with different phrasing. "Show me last month's revenue" and "What was total revenue in March?" are semantically identical but generate different SQL — BigQuery's native 24hr cache won't match them.

**Alternatives considered:**

| Option | Rejected because |
|---|---|
| BigQuery native cache only | Only matches byte-identical SQL strings |
| No cache | Every question calls LLM + hits BigQuery — high latency and cost |
| DynamoDB TTL | Additional non-GCP dependency; Redis has richer data structures |

**Rationale:** Keying the cache on the question's semantic hash (embedding similarity threshold > 0.95) catches paraphrased repeats. Cache hit = < 50ms response, zero LLM cost, zero BigQuery scan cost. At scale, expected cache hit rate > 40% for typical business users who ask similar questions.

**Trade-off:** The semantic hash approach can produce false positives for near-identical questions with different intent. Configurable similarity threshold allows tuning precision vs. recall.

---

## DEC-005 — Query Validator is Non-Negotiable

**Decision:** Every LLM-generated SQL statement must pass the Query Validator before execution. No bypass path exists.

**Context:** BigQuery charges per bytes scanned. A `SELECT *` on a 1 TB table costs ~$5.00 per query. A malicious or confused user could ask a question that generates a runaway query and exhaust a tenant's monthly budget in minutes.

**Alternatives considered:**

| Option | Rejected because |
|---|---|
| Trust LLM output directly | LLMs hallucinate and make mistakes; cost risk is unacceptable |
| Post-execution cost check | Too late — charge already incurred |
| BigQuery slot reservations | Caps execution speed but not scan cost |

**Rationale:** The validator is a cheap, deterministic check (regex + AST parse) that runs in < 5ms. It blocks: `SELECT *`, DDL/DML statements, cross-tenant table references, queries estimated to scan > 10 GB (configurable per tier). Blocked queries are returned to the NL2SQL Engine for retry with the specific violation as feedback. *This decision has zero exceptions.*

**Trade-off:** Some legitimate complex queries may be blocked by the scan size limit. Tenant admins can adjust the threshold per their cost tolerance.

---

## DEC-006 — Dual RAG + NL2SQL Routing

**Decision:** The Agent Orchestrator routes each question to either the NL2SQL pipeline or the RAG retriever — not both — based on intent classification.

**Context:** Users ask two fundamentally different types of questions:
1. "What were total sales last quarter?" → needs live database data → NL2SQL
2. "What does churn rate mean?" → needs business knowledge → RAG

**Alternatives considered:**

| Option | Rejected because |
|---|---|
| Always NL2SQL | Can't answer knowledge questions from SQL; confusing errors |
| Always RAG | Can't return live data; answers are always from static documents |
| Always both paths | 2x LLM calls and latency for every question |

**Rationale:** A fast LLM intent classifier (< 200ms) routes to the appropriate path. The classifier uses a small, cheap model (not the full reasoning model) to minimise overhead. Ambiguous questions fall back to NL2SQL as the primary capability. *[REF-003 Quivr](../references.md#ref-003) provides the RAG path defaults; [REF-001 Vanna.ai](../references.md#ref-001) provides the NL2SQL path defaults.*

**Trade-off:** Misclassification sends a data question to RAG (useless answer) or a knowledge question to NL2SQL (SQL error). Classifier accuracy is monitored and retrainable. Fallback: if NL2SQL fails after 3 retries, the Orchestrator escalates to RAG automatically.

---

## DEC-007 — Multi-tenant via BigQuery Dataset Isolation

**Decision:** Each client tenant gets a dedicated BigQuery Dataset with Row Level Security as a second enforcement layer.

**Context:** Multiple clients' data lives in the same GCP project. A misconfigured query or application bug must not allow one tenant to read another's data.

**Alternatives considered:**

| Option | Rejected because |
|---|---|
| Single shared dataset, filter by tenant_id column | Single bug in query generation exposes all tenants |
| Separate GCP project per tenant | Operational overhead scales linearly with tenant count |
| Database-level encryption only | Encryption doesn't prevent authorised app bugs from cross-reading |

**Rationale:** Dataset-level isolation means the service account for Tenant A literally cannot read Tenant B's Dataset — it's a GCP IAM boundary, not just an application-level filter. Row Level Security adds a second enforcement layer within the same Dataset for cases where tenants share a Dataset (cost optimisation for small tenants). Defense in depth: both layers must fail for cross-tenant data exposure.

**Trade-off:** Dataset-per-tenant adds management complexity when onboarding a new client. Mitigated by a tenant provisioning script that creates the Dataset, grants IAM, and registers the schema in Dataplex automatically.
