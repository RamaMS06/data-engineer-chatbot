# Integration & Runtime Specifications

## Integration

- Single `<script>` tag activation
- Config via HTML `data-*` attributes
- Shadow DOM for CSS isolation
- PostMessage API for iframe ↔ parent communication

## Serverless & Cost

- Auto-sleep after 20 minutes idle
- Cold start target < 2 seconds
- Pay-per-invocation pricing model
- $0 cost when system is inactive

## Database Support

| Database | Type |
|---|---|
| PostgreSQL | Relational |
| MySQL | Relational |
| SQLite | Embedded |
| BigQuery | Cloud DWH |
| Snowflake | Cloud DWH |
| Redshift | Cloud DWH |

Adapter pattern used for extensibility. Automatic schema introspection on connection.
