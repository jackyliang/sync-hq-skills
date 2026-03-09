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
Create sync states. **Auto-schedules syncs** with appropriate intervals based on resource type. Request: `{"connection_id": "uuid", "resources": ["tickets", "users"]}`

Sync states are created with `schedule_enabled=True` and:
- **2-hour interval** for `tickets` and `articles` (scheduler is fallback — webhooks handle realtime)
- **24-hour interval** for `sections` and `categories` (rarely change)

Response (202):
```json
{
  "message": "Sync triggered for 2 resource(s)",
  "sync_states": [{"sync_state_id": "uuid", "resource": "tickets", "status": "idle"}, ...]
}
```

**Delete handling (BYOP):** Tickets deleted in Zendesk are detected via the incremental export (`status: "deleted"`) and automatically removed during sync. Articles use a **daily reconciliation job** — Zendesk's incremental articles API omits deleted articles, so sync_hq fetches all live article IDs, diffs against the DB, and hard-deletes stale records.

#### POST /v1/syncs/{sync_state_id}/trigger
Trigger sync (202, background). Returns 409 if already running.

#### GET /v1/syncs/{sync_state_id}/status
Sync status with recent runs.
```json
{
  "sync_state_id": "uuid", "status": "idle", "last_cursor": "...", "retry_count": 0,
  "recent_runs": [{"id": "uuid", "status": "completed", "records_fetched": 150, "records_written": 150, "records_deleted": 0, "trigger_type": "webhook"}]
}
```

#### GET /v1/syncs
List syncs. Query params: `end_user_id`, `status`, `limit` (50), `offset` (0).

#### PUT /v1/syncs/{sync_state_id}/schedule
Override auto-sync schedule. Syncs are auto-scheduled on creation (see POST /v1/syncs), but this endpoint lets you customize. Request: `{"schedule_enabled": true, "interval_minutes": 60}`

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

## Webhook Ingest (Nango Forwarding)

#### POST /v1/webhooks/nango
Receive forwarded webhooks from Nango. **No API key required** — uses HMAC-SHA256 signature verification via `X-Nango-Signature` header.

When Zendesk data changes, Nango forwards the webhook here. sync_hq verifies the signature, maps the event to a resource (tickets/articles), and triggers an incremental sync in the background.

**Headers:** `X-Nango-Signature: <hmac-sha256-hex>`

**Request body (from Nango):**
```json
{
  "from": "zendesk",
  "type": "forward",
  "connectionId": "nango-connection-id",
  "providerConfigKey": "zendesk",
  "payload": {"type": "ticket", "id": 123}
}
```

**Responses:**
- `{"status": "accepted", "resources": ["tickets"]}` — sync triggered
- `{"status": "debounced", "reason": "..."}` — skipped (running or recently synced)
- `{"status": "ignored", "reason": "..."}` — no mapping or unknown connection
- `401` — invalid/missing signature

**Debouncing:** Skips if resource is already syncing or was synced within last 60s. Scheduler fallback (every 2hr) catches anything missed.

---

## Outbound Webhook Endpoints (Svix)

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
