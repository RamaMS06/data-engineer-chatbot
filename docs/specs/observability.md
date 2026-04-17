# Observability Specifications

All services emit structured JSON logs to GCP Cloud Logging. Every request produces a single root log entry with nested sub-entries per pipeline step.

---

## Log Structure

Every `/v1/chat` request produces:

```json
{
  "request_id": "req_01JGH3...",
  "tenant_id": "acme-corp",
  "session_id": "sess_9XK...",
  "timestamp": "2025-04-17T18:00:00Z",
  "pipeline": {
    "cache": {"hit": false, "latency_ms": 3},
    "intent_classify": {"result": "nl2sql", "latency_ms": 180},
    "schema_fetch": {"latency_ms": 12, "source": "redis"},
    "llm_generate": {"model": "gemini-1.5-pro", "tokens_in": 820, "tokens_out": 145, "latency_ms": 1240},
    "sql_validate": {"passed": true, "latency_ms": 4},
    "bq_execute": {"bytes_scanned": 1048576, "cost_usd": 0.005, "latency_ms": 890},
    "format": {"type": "table", "latency_ms": 210}
  },
  "total_latency_ms": 2542,
  "status": "success"
}
```

---

## Signals & Alerts

### SPC-O-001 — Latency Tracking Per Pipeline Step

**What:** End-to-end latency and per-step latency for every request. Steps tracked: cache lookup, intent classification, schema fetch, LLM generation, SQL validation, BigQuery execution, response formatting.

**Why:** Latency regressions in any step are immediately visible without re-deploying with extra logging. Allows targeted optimisation (e.g., if schema fetch suddenly spikes, Redis may be down).

**Alert:** P95 end-to-end latency > 5 seconds over a 10-minute window → PagerDuty alert.

### SPC-O-002 — Token Usage Per Query / Per Tenant

**What:** Input and output token counts logged per LLM call, attributed to `tenant_id` and `request_id`. Aggregated into a daily token usage report per tenant.

**Why:** LLM API cost is proportional to token usage. Without per-tenant attribution, a single high-traffic tenant could consume disproportionate cost with no visibility.

**Alert:** A single tenant's daily token spend > $50 USD → email alert to platform admin.

**Dashboard metric:** Tokens / request (rolling 24hr average) — useful for detecting prompt bloat if the schema injection grows too large.

### SPC-O-003 — SQL Error Rate Monitoring

**What:** Percentage of NL2SQL requests that required at least one retry (validator rejection or BigQuery execution error). Tracked per tenant and globally.

**Why:** A sudden spike in SQL error rate indicates either a schema change that the registry hasn't picked up, a model regression, or a tenant asking queries the current schema can't support.

**Alert:** SQL retry rate > 20% over a 1-hour rolling window → Slack alert to engineering channel.

**Dashboard metric:** `sql_errors_total / nl2sql_requests_total` (hourly buckets).

### SPC-O-004 — Cost Anomaly Alerts Per Tenant Per Day

**What:** Cloud Monitoring budget alert on BigQuery bytes scanned, attributed per tenant. Evaluated every 5 minutes.

**Why:** BigQuery's cost is per bytes scanned. A single bad query (escaped validator, or schema change making the validator's estimate wrong) can generate a large unexpected charge.

**Alert thresholds (configurable per tenant tier):**

| Tier | Daily scan budget | Alert at |
|---|---|---|
| Free | 10 GB | 80% |
| Starter | 100 GB | 80% |
| Pro | 1 TB | 80% |
| Enterprise | Custom | Custom |

**Action on breach:** Tenant's query execution is automatically throttled (not blocked) and an alert is sent to both the platform admin and the tenant's registered admin email.

### SPC-O-005 — Cache Hit Rate

**What:** Ratio of Redis cache hits to total requests, tracked per tenant and globally.

**Why:** Cache hit rate is the most direct measure of Redis ROI. If hit rate drops below 20%, the TTL may be too short or questions are too varied to benefit from caching. If it's above 60%, the TTL could be extended to improve it further.

**Dashboard metric:** `cache_hits / (cache_hits + cache_misses)` — hourly rolling average.

---

## Tools

| Tool | Purpose |
|---|---|
| GCP Cloud Logging | Structured log ingestion, query, and retention |
| GCP Cloud Monitoring | Metrics dashboards, alerting policies |
| Cloud Trace | Distributed request tracing with waterfall view per pipeline step |
| BigQuery (log sink) | Long-term log archival for cost analysis and compliance |
