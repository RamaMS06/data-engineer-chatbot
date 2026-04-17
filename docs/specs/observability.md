# Observability Specifications

| Signal | What's tracked |
|---|---|
| Latency | Per pipeline step (auth, NL2SQL, BigQuery, formatter) |
| Token usage | Per query, per tenant |
| SQL error rate | Failed queries, retry count |
| Cost alerts | Per tenant per day threshold |

## Tools

- **GCP Monitoring** — Centralized logging and per-request tracing
- **Cloud Logging** — Structured logs from all Cloud Run services
- **Cost anomaly alerts** — Triggered when daily BigQuery spend exceeds threshold
