# sync_hq Integration Skill

Use this skill when integrating sync_hq into an application — connecting providers, triggering syncs, reading synced data, and configuring webhooks.

## Overview

sync_hq syncs third-party data (Zendesk, etc.) into Postgres so AI agents can query it directly.

**Key concepts:**
- **Developer** — Your app/organization. Has API keys and owns connections.
- **End User** — Your customer. Each end user connects their own provider account (e.g., their Zendesk instance).
- **Connection** — An OAuth link between an end user and a provider. Created via Nango-hosted OAuth flow.
- **Sync** — A data pipeline that fetches records from a provider and writes them to Postgres. Each sync targets one resource (e.g., "tickets") for one connection.
- **Resource** — A data type from a provider (e.g., tickets, users, organizations for Zendesk).

## Prerequisites

1. A sync_hq API key (get one via `POST /v1/developers/signup` or from the dashboard)
2. sync_hq base URL

**Environment variables your project should have:**
```
SYNC_HQ_API_URL=https://api.sync-hq.com   # or http://localhost:8000 for local dev
SYNC_HQ_API_KEY=sk_live_...
```

## Authentication

All API calls require the `X-API-Key` header:

```
X-API-Key: sk_live_...
```

Verify your API key works:

```bash
curl $SYNC_HQ_API_URL/v1/developers/me \
  -H "X-API-Key: $SYNC_HQ_API_KEY"
```

Expected response:
```json
{
  "id": "uuid",
  "email": "you@example.com",
  "name": "Your Name",
  "is_active": true
}
```

## Integration Workflow

### Step 1: Connect a Provider for an End User

Start the OAuth flow so your end user can authorize their provider account (e.g., Zendesk):

```bash
# 1a. Create an OAuth session
curl -X POST $SYNC_HQ_API_URL/v1/connections/oauth \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"end_user_id": "customer_123", "provider": "zendesk"}'
```

Response:
```json
{
  "connect_link": "https://connect.nango.dev/...",
  "session_token": "nango_session_...",
  "expires_at": "2026-03-01T00:00:00Z"
}
```

Redirect the end user to `connect_link`. They authorize their Zendesk account there. After they complete OAuth:

```bash
# 1b. Confirm the connection
curl -X POST $SYNC_HQ_API_URL/v1/connections/confirm \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"end_user_id": "customer_123", "provider": "zendesk"}'
```

Response:
```json
{
  "id": "uuid",
  "developer_id": "uuid",
  "end_user_id": "customer_123",
  "provider": "zendesk",
  "status": "active",
  "provider_account_id": "mycompany",
  "nango_connection_id": "dev_id:customer_123",
  "resources": null,
  "created_at": "2026-02-28T12:00:00",
  "updated_at": "2026-02-28T12:00:00"
}
```

Save the `id` (connection UUID) — you'll use it for syncs and data reads.

### Step 2: Set Up a Sync

```bash
# 2a. Create sync states for the resources you want
curl -X POST $SYNC_HQ_API_URL/v1/syncs \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"connection_id": "<connection_uuid>", "resources": ["tickets", "users"]}'
```

Response (202):
```json
{
  "message": "Sync triggered for 2 resource(s)",
  "connection_id": "uuid",
  "resources": ["tickets", "users"],
  "sync_states": [
    {"sync_state_id": "uuid-1", "resource": "tickets", "status": "idle"},
    {"sync_state_id": "uuid-2", "resource": "users", "status": "idle"}
  ]
}
```

```bash
# 2b. Trigger the first sync
curl -X POST $SYNC_HQ_API_URL/v1/syncs/<sync_state_id>/trigger \
  -H "X-API-Key: $SYNC_HQ_API_KEY"
```

Response (202):
```json
{
  "sync_state_id": "uuid",
  "resource": "tickets",
  "status": "accepted",
  "message": "Sync triggered in background"
}
```

```bash
# 2c. (Optional) Enable automatic scheduling
curl -X PUT $SYNC_HQ_API_URL/v1/syncs/<sync_state_id>/schedule \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"schedule_enabled": true, "interval_minutes": 60}'
```

Response:
```json
{
  "sync_state_id": "uuid",
  "schedule_enabled": true,
  "interval_minutes": 60
}
```

### Step 3: Read Synced Data

```bash
curl "$SYNC_HQ_API_URL/v1/data/<connection_uuid>/tickets?limit=100&since=2026-01-01T00:00:00Z" \
  -H "X-API-Key: $SYNC_HQ_API_KEY"
```

Response:
```json
{
  "connection_id": "uuid",
  "resource": "tickets",
  "records": [
    {
      "id": 12345,
      "subject": "Help with billing",
      "status": "open",
      "synced_at": "2026-02-28T12:05:00"
    }
  ]
}
```

### Step 4: Set Up Webhooks (Optional)

Get notified when syncs complete or fail:

```bash
curl -X POST $SYNC_HQ_API_URL/v1/settings/webhooks \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://yourapp.com/webhooks/sync", "description": "Sync events"}'
```

Response (201):
```json
{
  "id": "ep_...",
  "url": "https://yourapp.com/webhooks/sync",
  "description": "Sync events"
}
```

**Webhook event types:**
| Event | Trigger |
|-------|---------|
| `sync.completed` | Sync finished successfully |
| `sync.failed` | Sync failed |
| `connection.created` | New connection established |
| `connection.deleted` | Connection removed |

## API Reference

All endpoints require `X-API-Key` header unless noted otherwise.

### Developer Endpoints

#### POST /v1/developers/signup
Create a new developer account. **No auth required.**

Request:
```json
{"email": "you@example.com", "name": "Your Name"}
```

Response (201):
```json
{
  "developer": {"id": "uuid", "email": "you@example.com", "name": "Your Name", "is_active": true},
  "api_key": "sk_live_..."
}
```

#### GET /v1/developers/me
Get authenticated developer profile.

Response:
```json
{"id": "uuid", "email": "you@example.com", "name": "Your Name", "is_active": true}
```

#### GET /v1/developers/me/keys
List all API keys.

Response:
```json
[
  {"id": "uuid", "key_prefix": "sk_live_abc...", "name": "Default", "is_active": true, "created_at": "2026-02-28T12:00:00"}
]
```

#### POST /v1/developers/me/keys
Create an additional API key. The raw key is returned only once.

Request:
```json
{"name": "Production Key"}
```

Response (201):
```json
{"id": "uuid", "api_key": "sk_live_...", "name": "Production Key", "key_prefix": "sk_live_abc..."}
```

#### DELETE /v1/developers/me/keys/{key_id}
Deactivate an API key (soft delete).

Response:
```json
{"status": "deactivated", "key_id": "uuid"}
```

---

### Connection Endpoints

#### POST /v1/connections/oauth
Create an OAuth session for an end user to connect a provider.

Request:
```json
{"end_user_id": "customer_123", "provider": "zendesk"}
```

Response:
```json
{"connect_link": "https://connect.nango.dev/...", "session_token": "...", "expires_at": "..."}
```

#### POST /v1/connections/confirm
Confirm a connection after the end user completes OAuth. Idempotent — returns existing connection if already confirmed.

Request:
```json
{"end_user_id": "customer_123", "provider": "zendesk"}
```

Response:
```json
{
  "id": "uuid", "developer_id": "uuid", "end_user_id": "customer_123",
  "provider": "zendesk", "status": "active", "provider_account_id": "mycompany",
  "nango_connection_id": "dev_id:customer_123", "resources": null,
  "created_at": "...", "updated_at": "..."
}
```

#### GET /v1/connections
List connections with optional filtering and pagination.

Query params: `end_user_id`, `provider`, `status`, `limit` (default 50), `offset` (default 0)

Response:
```json
{
  "items": [ConnectionResponse, ...],
  "total": 42,
  "limit": 50,
  "offset": 0
}
```

#### GET /v1/connections/{end_user_id}/{provider}
Get a specific connection by end user and provider.

Response: `ConnectionResponse`

#### DELETE /v1/connections/{end_user_id}/{provider}
Delete a connection from both Nango and local DB.

Response:
```json
{"status": "deleted", "end_user_id": "customer_123", "provider": "zendesk"}
```

#### GET /v1/connections/{connection_id}/available-resources
Get syncable resources for a connection's provider.

Response:
```json
{"connection_id": "uuid", "provider": "zendesk", "resources": ["tickets", "users", "organizations"]}
```

#### POST /v1/connections/{connection_id}/test
Test if a connection's OAuth token is still healthy.

Response:
```json
{"healthy": true, "connection_id": "uuid"}
```

Or if unhealthy:
```json
{"healthy": false, "connection_id": "uuid", "error": "Token expired"}
```

#### PUT /v1/connections/{connection_id}/scope
Update the selected resources for a connection.

Request:
```json
{"resources": ["tickets", "users"]}
```

Response: `ConnectionResponse`

---

### Sync Endpoints

#### POST /v1/syncs
Create sync states for specified resources. Does not trigger the sync — use `/trigger` after.

Request:
```json
{"connection_id": "uuid", "resources": ["tickets", "users"]}
```

Response (202):
```json
{
  "message": "Sync triggered for 2 resource(s)",
  "connection_id": "uuid",
  "resources": ["tickets", "users"],
  "sync_states": [
    {"sync_state_id": "uuid", "resource": "tickets", "status": "idle"},
    {"sync_state_id": "uuid", "resource": "users", "status": "idle"}
  ]
}
```

#### POST /v1/syncs/{sync_state_id}/trigger
Manually trigger a sync. Returns 202 immediately — sync runs in background. Returns 409 if sync is already running.

Response (202):
```json
{"sync_state_id": "uuid", "resource": "tickets", "status": "accepted", "message": "Sync triggered in background"}
```

#### GET /v1/syncs/{sync_state_id}/status
Get detailed sync status including recent runs.

Response:
```json
{
  "sync_state_id": "uuid",
  "connection_id": "uuid",
  "resource": "tickets",
  "status": "idle",
  "last_cursor": "2026-02-28T12:00:00Z",
  "retry_count": 0,
  "last_error": null,
  "recent_runs": [
    {
      "id": "uuid",
      "status": "completed",
      "started_at": "2026-02-28T12:00:00",
      "finished_at": "2026-02-28T12:01:00",
      "records_fetched": 150,
      "records_written": 150,
      "error_message": null
    }
  ]
}
```

#### GET /v1/syncs
List sync states with optional filtering and pagination.

Query params: `end_user_id`, `status`, `limit` (default 50), `offset` (default 0)

Response:
```json
{
  "items": [
    {
      "sync_state_id": "uuid", "connection_id": "uuid", "resource": "tickets",
      "status": "idle", "last_cursor": "...", "provider": "zendesk", "end_user_id": "customer_123"
    }
  ],
  "total": 10,
  "limit": 50,
  "offset": 0
}
```

#### PUT /v1/syncs/{sync_state_id}/schedule
Enable or disable automatic scheduling for a sync.

Request:
```json
{"schedule_enabled": true, "interval_minutes": 60}
```

Response:
```json
{"sync_state_id": "uuid", "schedule_enabled": true, "interval_minutes": 60}
```

#### GET /v1/syncs/{sync_state_id}/logs
Get logs for a sync's runs.

Query params: `level` (filter by log level), `limit` (default 100), `offset` (default 0)

Response:
```json
[
  {
    "id": "uuid",
    "sync_run_id": "uuid",
    "level": "info",
    "event": "fetch_started",
    "message": "Fetching tickets from Zendesk",
    "timestamp": "2026-02-28T12:00:00",
    "extra_data": null
  }
]
```

#### DELETE /v1/syncs/{sync_state_id}
Delete a sync state.

Response:
```json
{"status": "deleted", "sync_state_id": "uuid"}
```

---

### Data Endpoints

#### GET /v1/data/{connection_id}/{resource}
Read synced records for a connection's resource.

Query params: `since` (ISO 8601 timestamp, filters by synced_at), `limit` (default 100, max 1000), `offset` (default 0)

Response:
```json
{
  "connection_id": "uuid",
  "resource": "tickets",
  "records": [
    {"id": 12345, "subject": "Help with billing", "status": "open", "synced_at": "2026-02-28T12:05:00"}
  ]
}
```

Returns empty `records` array if no data has been synced yet (not an error).

---

### Webhook Endpoints

#### POST /v1/settings/webhooks
Register a webhook endpoint.

Request:
```json
{"url": "https://yourapp.com/webhooks/sync", "description": "Sync events"}
```

Response (201):
```json
{"id": "ep_...", "url": "https://yourapp.com/webhooks/sync", "description": "Sync events"}
```

#### GET /v1/settings/webhooks
List all webhook endpoints.

Response:
```json
[
  {"id": "ep_...", "url": "https://yourapp.com/webhooks/sync", "description": "Sync events"}
]
```

#### DELETE /v1/settings/webhooks/{endpoint_id}
Delete a webhook endpoint.

Response:
```json
{"status": "deleted"}
```

#### GET /v1/settings/webhooks/portal
Get a one-time Svix dashboard URL for managing webhooks in a web UI.

Response:
```json
{"portal_url": "https://app.svix.com/..."}
```

---

### Admin Endpoints

#### GET /v1/admin/health
Aggregate health overview for your connections and syncs.

Response:
```json
{
  "total_connections": 25,
  "active_connections": 23,
  "error_connections": 2,
  "syncs_by_status": {"idle": 40, "running": 3, "failed": 2},
  "recent_failures": [
    {"end_user_id": "customer_456", "provider": "zendesk", "resource": "tickets", "error": "Rate limited"}
  ]
}
```

---

### Infrastructure Endpoints

#### GET /health
Basic health check. **No auth required.**

Response: `{"status": "ok"}`

#### GET /ready
Readiness check (verifies database connectivity). **No auth required.**

Response: `{"status": "ready", "database": "connected"}`

## Common Recipes

### Connect Zendesk for a customer and start syncing tickets

```bash
# 1. Start OAuth
curl -X POST $SYNC_HQ_API_URL/v1/connections/oauth \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"end_user_id": "customer_123", "provider": "zendesk"}'
# → Redirect customer to connect_link

# 2. After OAuth completes, confirm
curl -X POST $SYNC_HQ_API_URL/v1/connections/confirm \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"end_user_id": "customer_123", "provider": "zendesk"}'
# → Save the connection id

# 3. Create sync for tickets
curl -X POST $SYNC_HQ_API_URL/v1/syncs \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"connection_id": "<connection_id>", "resources": ["tickets"]}'
# → Save the sync_state_id

# 4. Trigger the initial sync
curl -X POST $SYNC_HQ_API_URL/v1/syncs/<sync_state_id>/trigger \
  -H "X-API-Key: $SYNC_HQ_API_KEY"

# 5. Enable hourly auto-sync
curl -X PUT $SYNC_HQ_API_URL/v1/syncs/<sync_state_id>/schedule \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"schedule_enabled": true, "interval_minutes": 60}'
```

### Check if all connections are healthy

```bash
# Option A: Use admin health endpoint for overview
curl $SYNC_HQ_API_URL/v1/admin/health \
  -H "X-API-Key: $SYNC_HQ_API_KEY"

# Option B: Test a specific connection
curl -X POST $SYNC_HQ_API_URL/v1/connections/<connection_id>/test \
  -H "X-API-Key: $SYNC_HQ_API_KEY"
```

### Read recently synced data after a webhook notification

When your webhook handler receives a `sync.completed` event:

```bash
# Read data synced in the last hour
curl "$SYNC_HQ_API_URL/v1/data/<connection_id>/tickets?since=2026-02-28T11:00:00Z&limit=500" \
  -H "X-API-Key: $SYNC_HQ_API_KEY"
```

### Set up hourly sync with webhook notifications

```bash
# 1. Register a webhook endpoint
curl -X POST $SYNC_HQ_API_URL/v1/settings/webhooks \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://yourapp.com/webhooks/sync", "description": "Sync notifications"}'

# 2. Enable scheduling on each sync state
curl -X PUT $SYNC_HQ_API_URL/v1/syncs/<sync_state_id>/schedule \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"schedule_enabled": true, "interval_minutes": 60}'

# Now: sync runs every hour → webhook fires on completion/failure → your app reacts
```

## Error Handling

### HTTP Status Codes

| Code | Meaning |
|------|---------|
| 200 | Success |
| 201 | Created (signup, webhook creation, key creation) |
| 202 | Accepted (sync triggered — runs in background) |
| 400 | Bad request (invalid parameters, invalid resources) |
| 401 | Unauthorized (missing or invalid API key) |
| 404 | Not found (connection, sync state, or key doesn't exist) |
| 409 | Conflict (email already registered, sync already running) |
| 429 | Rate limited (100 requests/minute default) |
| 502 | Provider/upstream error (Nango API failure) |
| 503 | Service not ready (database not connected) |

### Common Errors

**401 — Bad API key:**
Verify the key starts with `sk_live_` and is passed via `X-API-Key` header (not `Authorization: Bearer`).

**404 — Connection not found:**
Either the connection doesn't exist, or it belongs to a different developer. Check `connection_id` and ensure you confirmed the connection after OAuth.

**409 — Sync already running:**
Wait for the current sync to complete before triggering again. Check status with `GET /v1/syncs/{sync_state_id}/status`.

**502 — Nango API error:**
The upstream OAuth provider or Nango is having issues. Check that the end user's OAuth token hasn't been revoked. Test the connection with `POST /v1/connections/{connection_id}/test`.

## Available Providers and Resources

| Provider | Resources |
|----------|-----------|
| `zendesk` | `tickets`, `users`, `organizations` |
