# sync_hq API Reference

All endpoints require `X-API-Key` header unless noted.

## Auth Endpoints (WorkOS AuthKit)

#### GET /v1/auth/login
Get WorkOS authorization URL. **No auth required.** Redirect user to the returned URL.

#### GET /v1/auth/callback
Handle OAuth callback. Creates user + developer + API key on first login. **No auth required.**

#### GET /v1/auth/me
Get current user + org memberships. **Requires session cookie.**

#### POST /v1/auth/logout
Clear session cookie.

---

## Developer Endpoints

#### GET /v1/developers/me
Get developer profile. Response: `{"id": "uuid", "email": "...", "name": "...", "is_active": true}`

#### GET /v1/developers/me/keys
List API keys. Response: `[{"id": "uuid", "key_prefix": "sk_live_abc...", "name": "Default", "is_active": true}]`

#### POST /v1/developers/me/keys
Create API key. Request: `{"name": "Production Key"}`. Response (201): `{"id": "uuid", "api_key": "sk_live_...", ...}`

#### DELETE /v1/developers/me/keys/{key_id}
Deactivate API key. Response: `{"status": "deactivated", "key_id": "uuid"}`

---

## Connection Endpoints

#### POST /v1/connections/oauth
Create OAuth session for end user.

Request: `{"end_user_id": "customer_123", "provider": "zendesk"}`

Response: `{"connect_link": "https://connect.nango.dev/...", "session_token": "...", "expires_at": "..."}`

#### POST /v1/connections/confirm
Confirm connection after OAuth. Idempotent.

Request: `{"end_user_id": "customer_123", "provider": "zendesk"}`

Response: `{"id": "uuid", "developer_id": "uuid", "end_user_id": "customer_123", "provider": "zendesk", "status": "active", ...}`

#### GET /v1/connections
List connections. Query params: `end_user_id`, `provider`, `status`, `limit` (50), `offset` (0).

Response: `{"items": [...], "total": 42, "limit": 50, "offset": 0}`

#### GET /v1/connections/{end_user_id}/{provider}
Get specific connection.

#### DELETE /v1/connections/{end_user_id}/{provider}
Delete connection from Nango and local DB. **Also drops all synced resource tables from the BYOP database.** If no other connections share the same schema (same developer + end_user), the schema is dropped too. The connection is marked as `deleting` during cleanup to block new connections for the same end_user+provider until cleanup completes. Response: `{"status": "deleted", ...}`

#### GET /v1/connections/{connection_id}/available-resources
List syncable resources. Response: `{"resources": ["tickets", "users", "organizations"]}`

#### POST /v1/connections/{connection_id}/test
Test OAuth token health. Response: `{"healthy": true}` or `{"healthy": false, "error": "..."}`

#### PUT /v1/connections/{connection_id}/scope
Update selected resources. Request: `{"resources": ["tickets", "users"]}`. Response: ConnectionResponse.

#### GET /v1/connections/events
List OAuth lifecycle events. Use to monitor when end users start, complete, or fail connecting their accounts.

Query params: `end_user_id`, `provider`, `event` (`oauth_started`, `oauth_completed`, `oauth_failed`), `limit` (50), `offset` (0).

Response:
```json
{
  "items": [
    {"id": "uuid", "end_user_id": "customer_123", "provider": "zendesk", "event": "oauth_failed", "message": "Connection not found in Nango", "created_at": "..."}
  ],
  "total": 5, "limit": 50, "offset": 0
}
```

---

## Sync Endpoints

#### POST /v1/syncs
Create sync states. Request: `{"connection_id": "uuid", "resources": ["tickets", "users"]}`

Response (202):
```json
{
  "message": "Sync triggered for 2 resource(s)",
  "sync_states": [{"sync_state_id": "uuid", "resource": "tickets", "status": "idle"}, ...]
}
```

#### POST /v1/syncs/{sync_state_id}/trigger
Trigger sync (202, background). Returns 409 if already running.

#### GET /v1/syncs/{sync_state_id}/status
Sync status with recent runs.
```json
{
  "sync_state_id": "uuid", "status": "idle", "last_cursor": "...", "retry_count": 0,
  "recent_runs": [{"id": "uuid", "status": "completed", "records_fetched": 150, "records_written": 150}]
}
```

#### GET /v1/syncs
List syncs. Query params: `end_user_id`, `status`, `limit` (50), `offset` (0).

#### PUT /v1/syncs/{sync_state_id}/schedule
Enable auto-sync. Request: `{"schedule_enabled": true, "interval_minutes": 60}`

#### GET /v1/syncs/{sync_state_id}/logs
Sync run logs. Query params: `level`, `limit` (100), `offset` (0).
```json
[{"id": "uuid", "level": "info", "event": "fetch_started", "message": "...", "timestamp": "..."}]
```

#### DELETE /v1/syncs/{sync_state_id}
Delete sync state.

---

## Database Settings Endpoints (BYOP)

#### PUT /v1/settings/database
Set custom Postgres URL. Encrypted at rest. Returns 409 if URL already set with active connections.

Request: `{"url": "postgresql://user:pass@host:5432/db"}`

Response: `{"configured": true}`

#### GET /v1/settings/database
Check if BYOP is configured. Response: `{"configured": true}` or `{"configured": false}`

#### DELETE /v1/settings/database
Remove custom database config. Returns 409 if active connections exist (must delete them first).

Response: `{"status": "cleared"}`

#### POST /v1/settings/database/test
Test connectivity and write permissions on a Postgres URL without saving it.

Request: `{"url": "postgresql://user:pass@host:5432/db"}`

Response: `{"success": true, "message": "Connection successful, write permissions verified"}` or `{"success": false, "message": "..."}`

---

## Webhook Endpoints

#### POST /v1/settings/webhooks
Register webhook. Request: `{"url": "https://...", "description": "..."}`. Response (201): `{"id": "ep_...", "url": "..."}`

#### GET /v1/settings/webhooks
List webhooks.

#### DELETE /v1/settings/webhooks/{endpoint_id}
Delete webhook.

#### GET /v1/settings/webhooks/portal
Get Svix dashboard URL. Response: `{"portal_url": "https://app.svix.com/..."}`

---

## Admin Endpoints

#### GET /v1/admin/health
Aggregate health overview.
```json
{
  "total_connections": 25, "active_connections": 23, "error_connections": 2,
  "syncs_by_status": {"idle": 40, "running": 3, "failed": 2},
  "recent_failures": [{"end_user_id": "...", "provider": "zendesk", "error": "Rate limited"}]
}
```

---

## Infrastructure

#### GET /health
Health check. No auth. Response: `{"status": "ok"}`

#### GET /ready
Readiness check. No auth. Response: `{"status": "ready", "database": "connected"}`
