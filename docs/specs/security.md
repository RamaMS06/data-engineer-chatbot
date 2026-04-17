# Security Specifications

| Control | Implementation |
|---|---|
| Auth per request | JWT validation |
| SQL injection prevention | Query Validator pre-execution |
| Tenant data isolation | Row-level security per tenant |
| Rate limiting | Per tenant-key |
| Secret management | AWS Secrets Manager / Vault |

## Multi-tenant Isolation Strategy

Each client gets a **separate BigQuery Dataset** with Row Level Security as a second layer of defense. DB credentials are resolved at runtime from Secrets Manager — never stored in application config.
