---
name: sync-hq
description: Integrates sync_hq into applications to sync third-party data (Zendesk, etc.) into Postgres. Use when connecting providers, triggering syncs, reading synced data, configuring BYOP databases, or setting up webhooks.
---

# sync_hq Integration

sync_hq syncs third-party data (Zendesk, etc.) into Postgres so AI agents can query it directly.

**Key concepts:**
- **Developer** — Your app/organization. Has API keys and owns connections.
- **End User** — Your customer. Each connects their own provider account (e.g., their Zendesk).
- **Connection** — An OAuth link between an end user and a provider.
- **Sync** — A data pipeline that fetches records from a provider and writes them to Postgres.

**Two data modes:**
- **Hosted (default)** — Synced data lives in sync_hq's cloud. Read it via API.
- **BYOP (Bring Your Own Postgres)** — Synced data writes to YOUR database. Query it natively with joins, pgvector, full-text search. See [byop-guide.md](byop-guide.md).

**IMPORTANT:** Before starting integration work, ask the user which mode they want:
> "Do you want synced data stored in sync_hq's hosted database (read via API), or written directly to your own Postgres (BYOP)?"
>
> - **Hosted** → Follow the standard workflow below. Data read via `GET /v1/data/...`.
> - **BYOP** → Set up your database first (see [byop-guide.md](byop-guide.md)), then follow the same workflow. Data queried directly from your Postgres.

## Prerequisites

1. Log in via WorkOS AuthKit (`GET /v1/auth/login`) — account + API key created on first login
2. Set environment variables:

```
SYNC_HQ_API_URL=https://api.sync-hq.com   # or http://localhost:8000
SYNC_HQ_API_KEY=sk_live_...
```

All API calls use `X-API-Key: sk_live_...` header.

## Integration Workflow

### Step 1: Connect a Provider

```bash
# Start OAuth flow
curl -X POST $SYNC_HQ_API_URL/v1/connections/oauth \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"end_user_id": "customer_123", "provider": "zendesk"}'
# → Returns { connect_link, session_token, expires_at }
# → Redirect end user to connect_link
```

After OAuth completes:

```bash
# Confirm the connection
curl -X POST $SYNC_HQ_API_URL/v1/connections/confirm \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"end_user_id": "customer_123", "provider": "zendesk"}'
# → Returns connection with id — save it
```

### Step 2: Create and Trigger a Sync

```bash
# Create sync states (auto-scheduled — no manual schedule step needed)
curl -X POST $SYNC_HQ_API_URL/v1/syncs \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"connection_id": "<connection_id>", "resources": ["tickets", "articles"]}'

# Trigger initial sync (runs in background)
curl -X POST $SYNC_HQ_API_URL/v1/syncs/<sync_state_id>/trigger \
  -H "X-API-Key: $SYNC_HQ_API_KEY"
```

**Sync triggers:** Tickets and articles sync via two mechanisms:
- **Webhook (primary)** — Nango forwards Zendesk webhooks to `POST /v1/webhooks/nango`, triggering near-realtime syncs when data changes
- **Scheduler (fallback)** — every 2 hours as a safety net

Sections and categories sync via scheduler only (every 24 hours).

Each sync run records its `trigger_type`: `webhook`, `scheduled`, or `manual`.

**Deletion detection:**
- **Tickets** — Zendesk's incremental API includes deleted tickets (`status: "deleted"`), automatically removed during sync
- **Articles** — Zendesk's incremental API omits deleted/archived articles with no signal. A **daily reconciliation job** fetches all live article IDs from Zendesk, diffs against the synced DB, and hard-deletes stale records. Skips if Zendesk returns 0 articles (safety guard).

To override the default schedule, use `PUT /v1/syncs/{sync_state_id}/schedule`:

```bash
curl -X PUT $SYNC_HQ_API_URL/v1/syncs/<sync_state_id>/schedule \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"schedule_enabled": true, "interval_minutes": 30}'
```

**Incremental sync:** Tickets and articles use incremental sync — after the initial full fetch, subsequent syncs only retrieve records that changed since the last sync. This is faster and uses fewer API calls. Sections and categories always do a full fetch.

### Step 3: Read Synced Data

**Hosted mode** — use the data read API:
```bash
curl "$SYNC_HQ_API_URL/v1/data/<connection_id>/tickets?limit=100&since=2026-01-01T00:00:00Z" \
  -H "X-API-Key: $SYNC_HQ_API_KEY"
```

**BYOP mode** — query your own Postgres directly:
```sql
SELECT id, subject, status, created_at::timestamptz
FROM sync_abcd1234_customer_123.tickets
WHERE _synced_at > NOW() - INTERVAL '1 hour';
```

For BYOP setup, schema naming, and query patterns, see [byop-guide.md](byop-guide.md).

### Step 4: Set Up Webhooks

```bash
curl -X POST $SYNC_HQ_API_URL/v1/settings/webhooks \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://yourapp.com/webhooks/sync", "description": "Sync events"}'
```

Events: `sync.completed`, `sync.failed`, `connection.created`, `connection.deleted`

## Data Schema

Synced data lands in per-end-user schemas:
```
sync_{dev_prefix}_{end_user_id}.tickets
sync_{dev_prefix}_{end_user_id}.articles
sync_{dev_prefix}_{end_user_id}.sections
sync_{dev_prefix}_{end_user_id}.categories
```

All columns are TEXT except `id` (PK) and `_synced_at` (TIMESTAMPTZ, auto-set). Cast when querying typed data.

### Step 5: Disconnect a Provider

```bash
curl -X DELETE $SYNC_HQ_API_URL/v1/connections/{end_user_id}/{provider} \
  -H "X-API-Key: $SYNC_HQ_API_KEY"
```

**This automatically:**
1. Marks the connection as `deleting` (blocks new connections for same end_user+provider)
2. Drops all synced resource tables from the BYOP database
3. Drops the schema if no other connections share it
4. Deletes all sync metadata (sync_states, sync_runs, sync_logs)
5. Deletes the connection record

**No manual SQL cleanup needed.** All synced data is removed automatically.

**409 on reconnect:** If you try to confirm a new connection while a previous one is still being cleaned up, you'll get a `409 Conflict`. Retry shortly.

## Error Handling

| Code | Meaning |
|------|---------|
| 200/201/202 | Success / Created / Accepted (sync runs in background) |
| 400 | Bad request (BYOP users calling data read API) |
| 401 | Missing or invalid API key |
| 404 | Connection or sync not found |
| 409 | Conflict (sync already running, DB URL change blocked, or connection still being cleaned up) |
| 422 | Invalid input (bad Postgres URL) |
| 429 | Rate limited (100 req/min) |
| 502 | Provider/Nango error |

## Reference

- **Full API reference**: See [api-reference.md](api-reference.md)
- **BYOP setup and querying**: See [byop-guide.md](byop-guide.md)
- **Recipes and troubleshooting**: See [recipes.md](recipes.md)
- **Answer HQ integration**: See [answerhq-integration.md](answerhq-integration.md) — end-to-end flow from sync to AI search

## Choosing Resources to Sync

sync_hq focuses on syncing **knowledge-base content** — data that helps AI agents answer questions. Not every available resource is useful.

**When setting up syncs, ask the user which resources they actually need.** Present the available resources and help them choose based on their use case. Default to knowledge-oriented resources (tickets, articles, help center content) rather than syncing everything.

**Example prompt:**
> "Zendesk has these available resources: tickets, articles, sections, categories. Which ones does your AI agent need? For a support assistant, I'd recommend tickets and articles."

## Available Providers

| Provider | Resources |
|----------|-----------|
| `zendesk` | `tickets`, `articles`, `sections`, `categories` |
