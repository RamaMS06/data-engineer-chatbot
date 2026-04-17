# Security Specifications

---

## SPC-S-001 — JWT Per-Request Auth

**Control:** Every request to the backend API must carry a valid JWT in the `Authorization: Bearer <token>` header.

**How it works:**
```
 Widget init
   │
   ▼
 POST /v1/auth/token  {tenant_key: "..."}
   │
   ▼
 Auth Middleware validates tenant_key against Cloud SQL tenants table
   │
   ├── VALID   ──► issue JWT (15-min TTL, scoped to tenant_id)
   └── INVALID ──► 401 Unauthorized
```
The JWT payload contains: `{tenant_id, issued_at, expires_at, scope: "chat"}`. All downstream services (NL2SQL, RAG, BigQuery Connector) extract `tenant_id` from the JWT — they trust the token, not the original key.

**Why JWT over session cookies:** The widget runs cross-origin inside an iframe. Cookies require `SameSite=None; Secure` and are blocked by some browser privacy settings. JWTs in `Authorization` headers work universally.

---

## SPC-S-002 — SQL Injection Prevention

**Control:** The Query Validator ([ARC-L3-003](../architecture/overview.md#layer-3--ai-core-engine)) blocks dangerous SQL patterns before execution. Additional parameterised query enforcement at the DB connector layer.

**Validator rules (non-exhaustive):**

| Rule ID | Pattern blocked | Reason |
|---|---|---|
| SPC-S-002a | `SELECT *` | Cost: scans all columns, charges for full table |
| SPC-S-002b | `DROP`, `TRUNCATE`, `DELETE`, `UPDATE`, `INSERT` | DDL/DML not allowed — read-only access only |
| SPC-S-002c | Cross-dataset references | Prevents tenant A reading tenant B's Dataset |
| SPC-S-002d | `UNION` with external table | Prevents schema enumeration attacks |
| SPC-S-002e | Estimated scan > 10 GB | Cost cap; configurable per tenant tier |
| SPC-S-002f | Subquery depth > 5 | Prevents runaway query complexity |

The validator returns a structured error to the NL2SQL Engine with the specific rule violated, enabling targeted retry.

---

## SPC-S-003 — Row-Level Security Per Tenant

**Control:** BigQuery Row Level Security (RLS) policies are applied at the Dataset level for each tenant. Even if the application layer makes an error in query routing, the database enforces access boundaries.

**Implementation:**
```sql
-- Applied to every table in tenant_acme Dataset
CREATE ROW ACCESS POLICY acme_isolation
ON tenant_acme.orders
GRANT TO ("serviceAccount:acme-sa@project.iam.gserviceaccount.com")
FILTER USING (TRUE);
-- All other service accounts: FILTER USING (FALSE)
```

Defense in depth: Dataset-level IAM is the first boundary. RLS is the second. Both must fail for cross-tenant exposure. See [DEC-007](../planning/design-decisions.md#dec-007).

---

## SPC-S-004 — Rate Limiting Per Tenant

**Control:** Each tenant's `tenant_key` is subject to a request rate limit enforced at the API Gateway layer and tracked in Redis.

| Tier | Limit |
|---|---|
| Free | 20 requests / minute |
| Starter | 100 requests / minute |
| Pro | 500 requests / minute |
| Enterprise | Custom |

Rate limit counters are stored in Redis with a 1-minute rolling window. Requests exceeding the limit receive `429 Too Many Requests` with a `Retry-After` header.

---

## SPC-S-005 — Secret Management

**Control:** All credentials (database passwords, API keys, tenant secrets) are stored in GCP Secret Manager or HashiCorp Vault. Secrets are never hardcoded in source code, environment files, or container images.

**Access pattern:**
```
 Cloud Run service (runtime)
   │
   ▼
 Secret Manager API  ──► returns secret value at request time
   │
   ▼
 Injected as env variable into request context
   (not persisted in memory beyond the request lifecycle)
```

Secret rotation is automated for database credentials (rotation every 90 days). API keys are rotated manually by the tenant admin via dashboard. All secret access is logged in Cloud Audit Logs.

---

## SPC-S-006 — CORS Allowlist

**Control:** The API Gateway enforces a per-tenant CORS allowlist. Only requests originating from the tenant's registered domains are accepted.

Each tenant registers their allowed origins in the dashboard (e.g., `https://app.acme.com`). Requests from unregistered origins receive `403 Forbidden` even with a valid JWT. This prevents a stolen tenant-key from being used on an attacker-controlled page.
