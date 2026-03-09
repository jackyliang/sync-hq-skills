# sync_hq Recipes

## Connect Zendesk and start syncing

```bash
# 1. Start OAuth
curl -X POST $SYNC_HQ_API_URL/v1/connections/oauth \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"end_user_id": "customer_123", "provider": "zendesk"}'
# → Redirect customer to connect_link

# 2. Confirm after OAuth
curl -X POST $SYNC_HQ_API_URL/v1/connections/confirm \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"end_user_id": "customer_123", "provider": "zendesk"}'
# → Save connection id

# 3. Create sync (auto-scheduled: tickets every 5min, sections/categories every 60min)
curl -X POST $SYNC_HQ_API_URL/v1/syncs \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"connection_id": "<id>", "resources": ["tickets"]}'

# 4. Trigger initial sync
curl -X POST $SYNC_HQ_API_URL/v1/syncs/<sync_state_id>/trigger \
  -H "X-API-Key: $SYNC_HQ_API_KEY"

# Auto-scheduling is on by default. To customize:
# curl -X PUT $SYNC_HQ_API_URL/v1/syncs/<sync_state_id>/schedule \
#   -H "X-API-Key: $SYNC_HQ_API_KEY" \
#   -H "Content-Type: application/json" \
#   -d '{"schedule_enabled": true, "interval_minutes": 30}'
```

## Set up BYOP (Bring Your Own Postgres)

```bash
# 1. Test your Postgres URL
curl -X POST $SYNC_HQ_API_URL/v1/settings/database/test \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "postgresql://sync_hq_writer:pass@db.supabase.co:5432/mydb"}'

# 2. Save it (encrypted at rest)
curl -X PUT $SYNC_HQ_API_URL/v1/settings/database \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "postgresql://sync_hq_writer:pass@db.supabase.co:5432/mydb"}'

# 3. Register webhooks
curl -X POST $SYNC_HQ_API_URL/v1/settings/webhooks \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://yourapp.com/webhooks/sync-hq", "description": "Sync events"}'

# 4. Connect providers and sync as usual — data now lands in YOUR database
```

## Debugging a Failed Sync

When a sync fails (or you receive a `sync.failed` webhook):

```bash
# 1. Check sync status — see last error, retry count, recent runs
curl $SYNC_HQ_API_URL/v1/syncs/<sync_state_id>/status \
  -H "X-API-Key: $SYNC_HQ_API_KEY"

# 2. Read detailed logs — event-level detail
curl "$SYNC_HQ_API_URL/v1/syncs/<sync_state_id>/logs?level=error" \
  -H "X-API-Key: $SYNC_HQ_API_KEY"

# 3. Test connection health — is the OAuth token still valid?
curl -X POST $SYNC_HQ_API_URL/v1/connections/<connection_id>/test \
  -H "X-API-Key: $SYNC_HQ_API_KEY"

# 4. Check admin overview — see failures across all connections
curl $SYNC_HQ_API_URL/v1/admin/health \
  -H "X-API-Key: $SYNC_HQ_API_KEY"
```

Logs and sync metadata always live in sync_hq's cloud, even with BYOP.

**Webhook handler example:**

```python
@router.post("/webhooks/sync-hq")
async def handle_sync_event(payload: dict):
    match payload["type"]:
        case "sync.completed":
            schema = payload["schema_name"]
            resource = payload["resource"]
            await rebuild_search_index(schema, resource)

        case "sync.failed":
            # Fetch detailed error info
            status = await sync_hq_client.get(
                f"/v1/syncs/{payload['sync_state_id']}/status"
            )
            await alert_admin(
                end_user=payload["end_user_id"],
                error=payload["error"],
                retry_count=status["retry_count"],
            )

        case "connection.deleted":
            # Synced data is automatically dropped by sync_hq on disconnect.
            # Use this hook to clean up your own app state (e.g., remove search indexes).
            await on_connection_removed(payload["end_user_id"])
```

## Check connection health

```bash
# Overview of all connections
curl $SYNC_HQ_API_URL/v1/admin/health \
  -H "X-API-Key: $SYNC_HQ_API_KEY"

# Test a specific connection
curl -X POST $SYNC_HQ_API_URL/v1/connections/<connection_id>/test \
  -H "X-API-Key: $SYNC_HQ_API_KEY"
```

## Set up webhooks for sync events

Syncs are auto-scheduled on creation (tickets/articles every 5min, sections/categories every 60min). To receive notifications when syncs complete or fail, register a webhook:

```bash
# Register webhook
curl -X POST $SYNC_HQ_API_URL/v1/settings/webhooks \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://yourapp.com/webhooks/sync", "description": "Sync notifications"}'

# Syncs run on their auto-schedule → webhook fires → your app reacts
```

To customize the schedule for a specific sync, use `PUT /v1/syncs/<sync_state_id>/schedule`:

```bash
curl -X PUT $SYNC_HQ_API_URL/v1/syncs/<sync_state_id>/schedule \
  -H "X-API-Key: $SYNC_HQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"schedule_enabled": true, "interval_minutes": 30}'
```

## Deletion detection

sync_hq automatically detects and removes deleted records:

- **Tickets**: Zendesk's incremental API includes deleted tickets (`status: "deleted"`). These are hard-deleted from the synced DB during normal sync.
- **Articles**: Zendesk's incremental articles API does **not** include deleted/archived articles. A daily reconciliation job fetches all live article IDs from Zendesk, compares against the DB, and hard-deletes stale records. This runs automatically via the scheduler (~every 24 hours). A safety guard skips deletion if Zendesk returns 0 articles (prevents mass deletion on API errors).
