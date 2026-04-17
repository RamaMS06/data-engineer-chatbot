# Integration & Runtime Specifications

---

## Integration

### SPC-T-001 — Single Script Tag Activation
The entire widget is activated by adding one `<script>` tag to any HTML page. No npm install, no build step, no framework dependency. The client copies a single line from their dashboard and pastes it before `</body>`.

```html
<script
  src="https://cdn.aidataassistant.com/widget.js"
  data-tenant-id="acme-corp"
  data-theme="dark"
  data-db-schema="sales">
</script>
```

The loader script is served from a CDN (Cloud CDN via GCS) and is < 5 KB gzipped.

### SPC-T-002 — Config via HTML Data Attributes
All client-side configuration is passed declaratively via `data-*` attributes. No JavaScript API required.

| Attribute | Required | Default | Description |
|---|---|---|---|
| `data-tenant-id` | Yes | — | Client identifier, used to resolve credentials and schema |
| `data-theme` | No | `light` | Widget theme: `light`, `dark`, or a CSS hex colour |
| `data-db-schema` | No | `default` | Which schema namespace to query |
| `data-position` | No | `bottom-right` | Widget button position on the host page |
| `data-language` | No | `en` | Response language for the LLM |

### SPC-T-003 — Shadow DOM CSS Isolation
The widget UI renders inside a Shadow DOM attached to the iframe document root. The host page's CSS cannot reach inside the Shadow DOM boundary. The widget's CSS cannot leak out. This guarantees the widget looks identical on every host application regardless of their CSS framework. See [DEC-002](../planning/design-decisions.md#dec-002).

### SPC-T-004 — PostMessage API for iframe Communication
The Widget Loader JS and the Chat Widget UI (inside the iframe) communicate exclusively via the `window.postMessage` API with strict origin validation. No shared DOM access. Messages are typed: `{type: "CHAT_MESSAGE" | "RESIZE" | "CLOSE" | "THEME_UPDATE", payload: ...}`.

---

## Serverless & Cost

### SPC-T-005 — Auto-sleep After 20 Minutes Idle
Cloud Run services scale to zero replicas after 20 minutes with no incoming requests. When scaled to zero, the platform incurs $0 compute cost. This is the default behaviour — no configuration required. Tenants on paid tiers can opt into min-instances = 1 for lower cold start latency.

### SPC-T-006 — Cold Start Target < 2 Seconds
The FastAPI container is optimised for fast cold start:
- Slim Python base image (`python:3.12-slim`)
- Dependencies pre-installed at build time, not at startup
- LLM API clients initialised lazily (only on first request after cold start)
- Cloud Run startup CPU boost enabled (2x CPU during initialisation)

Target: < 2 seconds from container start to first byte served, measured via Cloud Run metrics.

### SPC-T-007 — Pay-per-invocation Pricing
Cost components:
- **Cloud Run:** billed per 100ms of CPU + memory used during request processing. Zero cost when scaled to zero.
- **BigQuery:** billed per bytes scanned. Query Validator ([ARC-L3-003]) enforces a scan budget per query.
- **Vertex AI:** billed per 1,000 tokens (input + output). Token count logged per request.
- **Redis:** fixed monthly cost for Memorystore instance (smallest tier: ~$30/month).

### SPC-T-008 — $0 Cost When Inactive
When no widget is actively being used (no requests for > 20 minutes): Cloud Run = $0, BigQuery = $0, Vertex AI = $0. Only Redis and Cloud SQL have fixed monthly costs. Target minimum monthly infrastructure cost at zero usage: < $50.

---

## Database Support

### SPC-T-009 — Adapter Pattern for Multi-DB Support
All database connections implement a common `DBAdapter` interface:

```python
class DBAdapter:
    def connect(credentials: dict) -> None: ...
    def execute(sql: str) -> QueryResult: ...
    def introspect_schema() -> SchemaMetadata: ...
    def estimate_scan_bytes(sql: str) -> int: ...
```

New database backends are added by implementing this interface — no changes to the NL2SQL Engine or Query Validator.

### SPC-T-010 — Supported Databases

| ID | Database | Type | Status |
|---|---|---|---|
| SPC-T-010a | BigQuery | Cloud DWH | Primary — fully supported |
| SPC-T-010b | PostgreSQL | Relational | Supported |
| SPC-T-010c | MySQL | Relational | Supported |
| SPC-T-010d | SQLite | Embedded | Dev/test only |
| SPC-T-010e | Snowflake | Cloud DWH | Planned |
| SPC-T-010f | Redshift | Cloud DWH | Planned |

### SPC-T-011 — Automatic Schema Introspection
On first connection to a new tenant's database, the Schema Loader ([TSK-004](../planning/roadmap.md#tsk-004--schema-registry)) automatically runs `introspect_schema()` and registers the result in Dataplex. No manual schema definition required from the client.
