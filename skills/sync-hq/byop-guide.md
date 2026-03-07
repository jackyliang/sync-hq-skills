# Bring Your Own Postgres (BYOP)

BYOP lets sync_hq write synced data directly into YOUR Postgres database. Query it natively — joins with your own tables, pgvector, pg_textsearch, hybrid search.

## Setup

### 1. Create a Postgres user for sync_hq

```sql
CREATE USER sync_hq_writer WITH PASSWORD 'secure_password';
GRANT CREATE ON DATABASE mydb TO sync_hq_writer;
-- sync_hq auto-creates schemas and tables. No other setup needed.
```

Works with any Postgres-compatible service: Supabase, Neon, RDS, etc. SSL recommended.

### 2. Register the database

```bash
# Test connectivity first
curl -X POST $SYNC_HQ_API_URL/v1/settings/database/test \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "postgresql://sync_hq_writer:pass@db.supabase.co:5432/mydb"}'

# If success, save it
curl -X PUT $SYNC_HQ_API_URL/v1/settings/database \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "postgresql://sync_hq_writer:pass@db.supabase.co:5432/mydb"}'
```

Once set, all syncs write to your database. The URL is encrypted at rest and cannot be changed while active connections exist.

### 3. Connect providers and sync (same as hosted)

The OAuth flow, sync creation, and scheduling are identical. Only the write target changes.

## Schema Naming

sync_hq creates schemas in your database using this formula:

```
sync_{first_8_chars_of_developer_id}_{end_user_id}
```

Sanitized: lowercase, non-alphanumeric replaced with `_`, truncated to 63 chars.

**Helper function for your app:**

```python
import re, os

SYNC_HQ_DEV_ID = os.environ["SYNC_HQ_DEV_ID"]  # from GET /v1/developers/me → id

def get_sync_schema(end_user_id: str) -> str:
    prefix = re.sub(r'[^a-z0-9_]', '_', SYNC_HQ_DEV_ID[:8].lower())
    end_user = re.sub(r'[^a-z0-9_]', '_', end_user_id.lower())
    return f"sync_{prefix}_{end_user}"[:63]
```

```typescript
const SYNC_HQ_DEV_ID = process.env.SYNC_HQ_DEV_ID!;

function getSyncSchema(endUserId: string): string {
  const prefix = SYNC_HQ_DEV_ID.slice(0, 8).toLowerCase().replace(/[^a-z0-9_]/g, '_');
  const endUser = endUserId.toLowerCase().replace(/[^a-z0-9_]/g, '_');
  return `sync_${prefix}_${endUser}`.slice(0, 63);
}
```

Get your developer_id: `GET /v1/developers/me` → `id` field.

## Table Structure

- Tables named after the resource: `tickets`, `users`, `organizations`
- All columns are **TEXT** — cast when querying typed data
- `id` TEXT is the primary key
- `_synced_at` TIMESTAMPTZ is auto-added (set on each upsert)
- Tables are dynamic: created on first sync, new columns may appear if the provider adds fields

**Sync behavior by resource:**

| Resource | Sync type | Interval | Deletes auto-removed? |
|----------|-----------|----------|-----------------------|
| `tickets` | Incremental (changes only) | 5 min | Yes |
| `articles` | Incremental (changes only) | 5 min | No |
| `sections` | Full fetch | 60 min | No |
| `categories` | Full fetch | 60 min | No |

- **Incremental sync** (tickets, articles): After the initial full fetch, only records that changed since the last sync are fetched. Much faster and uses fewer API calls.
- **Full fetch** (sections, categories): All records are fetched every time.
- **Delete handling** (tickets only): When a ticket is deleted in Zendesk, the incremental export includes it as deleted. sync_hq automatically removes the row from your BYOP table. Other resources do not currently detect deletes.

## Querying Synced Data

```sql
-- Basic query with type casting
SELECT id, subject, status, created_at::timestamptz, _synced_at
FROM sync_abcd1234_customer_123.tickets
WHERE _synced_at > NOW() - INTERVAL '1 hour';

-- Join with your own tables
SELECT t.subject, t.status, c.plan_name
FROM sync_abcd1234_customer_123.tickets t
JOIN customers c ON c.external_id = t.requester_id;
```

**Using webhooks with BYOP:** The `sync.completed` payload includes `schema_name` — use it directly in webhook handlers instead of recomputing it.

## Tracking Connections Locally

BYOP philosophy: your data, your database. Don't depend on sync_hq API calls at runtime for basic routing like "is this customer connected?"

**Create a local tracking table:**

```sql
CREATE TABLE sync_hq_connections (
  end_user_id TEXT,
  provider TEXT,
  sync_hq_connection_id UUID,
  status TEXT DEFAULT 'active',
  connected_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE (end_user_id, provider)
);
```

Insert when `POST /v1/connections/confirm` succeeds. Keep current via webhooks (`connection.created`, `connection.deleted`).

Hosted users can skip this — use `GET /v1/connections?end_user_id=X` instead.

## Disconnecting & Cleanup

`DELETE /v1/connections/{end_user_id}/{provider}` removes the connection **and automatically drops all synced resource tables** from the BYOP database. If no other connections share the schema (same developer + end_user), the entire schema is dropped too.

The connection is marked as `deleting` during cleanup. While in this state, `POST /v1/connections/confirm` for the same end_user+provider returns **409 Conflict**. This prevents race conditions where a new connection is created before cleanup finishes.

**Teardown sequence (handled automatically by the DELETE endpoint):**

1. Connection marked as `deleting` (blocks new connections)
2. Nango connection deleted (best-effort)
3. Synced resource tables dropped from BYOP database
4. Schema dropped if no other connections share it
5. sync_logs, sync_runs, sync_states deleted
6. Connection record deleted

**No manual SQL cleanup needed.** Just call:

```bash
curl -X DELETE $SYNC_HQ_API_URL/v1/connections/{end_user_id}/{provider} \
  -H "X-API-Key: $SYNC_HQ_API_KEY"
```

**Safety:** If the end user has multiple providers (Zendesk + another), they share the schema. Only drop it when ALL connections for that end user are removed.

**Automate via webhooks:** Handle `connection.deleted` to trigger cleanup automatically.

## Building on Synced Data

**Don't modify synced tables directly.** sync_hq upserts on every sync and will overwrite changes. Build derived tables alongside them.

**Recommended indexes:**

```sql
CREATE INDEX ON sync_abcd1234_customer_123.tickets (_synced_at);
CREATE INDEX ON sync_abcd1234_customer_123.tickets ((status::text));
```

**Typed views** (cast once, use everywhere):

```sql
CREATE VIEW customer_tickets AS
SELECT id, subject, status::text, priority::text,
       created_at::timestamptz, updated_at::timestamptz, _synced_at
FROM sync_abcd1234_customer_123.tickets;
```

**Full-text search:**

```sql
-- Add tsvector column and GIN index
ALTER TABLE sync_abcd1234_customer_123.tickets
  ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (to_tsvector('english', coalesce(subject, '') || ' ' || coalesce(description, ''))) STORED;

CREATE INDEX ON sync_abcd1234_customer_123.tickets USING GIN (search_vector);

-- Query
SELECT id, subject, ts_rank(search_vector, query) AS rank
FROM sync_abcd1234_customer_123.tickets, to_tsquery('english', 'billing & issue') query
WHERE search_vector @@ query
ORDER BY rank DESC LIMIT 10;
```

**Refresh strategy:** Use `sync.completed` webhook to trigger index/embedding rebuilds for new records (filter by `_synced_at > last_processed`).

## Where Data Lives

| Data | Where |
|------|-------|
| Synced provider data (tickets, users) | **Your Postgres** |
| Sync runs, logs, state | sync_hq cloud (always) |
| Connections, API keys | sync_hq cloud (always) |

Logs are always accessible via `GET /v1/syncs/{id}/logs` even with BYOP.
