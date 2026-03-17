# How Answer HQ + sync_hq Work Together

This is the single source of truth for how synced third-party data (Zendesk, etc.) becomes searchable by the AI assistant.

## The Big Picture

```
Zendesk Help Center
        |
        v
    sync_hq          (syncs articles into Postgres)
        |
        v
  BYOP Postgres      (raw synced data lives here)
        |
   webhook fires
        |
        v
    answerhq          (converts articles to searchable chunks)
        |
        v
  Supabase pgvector   (chunks + embeddings stored here)
        |
        v
    AI Assistant       (searches chunks to answer questions)
```

**sync_hq** owns syncing. It connects to Zendesk, fetches articles, and writes them as raw data into a BYOP Postgres database. When a sync finishes, it fires a webhook.

**answerhq** owns indexing. It receives the webhook, reads the raw articles from BYOP, converts HTML to markdown, splits into chunks, generates embeddings, and stores them in pgvector. The AI assistant then searches these chunks when answering questions.

Neither system knows the other's internals. They communicate through two touchpoints: the **BYOP database** (shared data) and the **webhook** (event notification).

## Step-by-Step Flow

### 1. User connects Zendesk (one-time setup)

The user goes to answerhq's dashboard (`/dashboard/connectors`), clicks "Connect Zendesk", and completes OAuth. Behind the scenes, answerhq proxies the OAuth to sync_hq's API.

### 2. sync_hq syncs articles

sync_hq fetches articles from Zendesk and writes them into the BYOP Postgres database. Each organization gets its own schema:

```
Schema: sync__171e499f__<org_id_with_underscores>
Tables: articles, tickets, sections, categories
```

Articles are stored as raw Zendesk data — the `body` column contains HTML from Zendesk's WYSIWYG editor.

Syncs happen automatically via two mechanisms:
- **Webhooks from Zendesk** (near-realtime when articles change)
- **Scheduled polling** (every 2 hours as a safety net)

### 3. sync_hq fires a webhook

When a sync completes, sync_hq sends a `sync.completed` webhook to answerhq:

```
POST https://answerhq.onrender.com/api/connectors/webhook
```

```json
{
  "connection_id": "uuid",
  "end_user_id": "the-org-id-in-answerhq",
  "resource": "articles",
  "provider": "zendesk",
  "sync_state_id": "uuid",
  "schema_name": "sync__171e499f__org_id",
  "records_fetched": 50,
  "records_written": 50,
  "completed_at": "2026-03-16T18:00:00Z"
}
```

**What each field is for:**

| Field | Purpose |
|-------|---------|
| `end_user_id` | The org ID — answerhq uses this to find which assistant to update |
| `resource` | What was synced — answerhq only indexes "articles" (ignores tickets, etc.) |
| `provider` | Which service — determines how to convert the content (Zendesk HTML vs. others) |
| `sync_state_id` | Unique sync identifier — used to tag chunks so re-indexing replaces the right ones |
| `connection_id` | The Zendesk connection — not used by answerhq currently |
| `schema_name` | The BYOP schema name — not used (answerhq builds it from `end_user_id`) |
| `records_fetched/written` | Stats — not used, informational only |
| `completed_at` | Timestamp — not used, informational only |

### 4. answerhq indexes the articles

When answerhq receives the webhook, it:

1. **Checks the resource** — only "articles" triggers indexing (tickets are ignored)
2. **Finds the assistant** — looks up `end_user_id` (org ID) to find the assistant
3. **Reads articles from BYOP** — connects to the BYOP database and reads all articles from the org's schema
4. **Converts HTML to markdown** — Zendesk article bodies are HTML; `html_to_markdown()` converts them, stripping images/scripts/iframes
5. **Chunks the text** — `chunk_text()` splits long articles into ~1000 token pieces, respecting markdown headers
6. **Generates embeddings** — sends chunks to OpenRouter's embedding API
7. **Stores in pgvector** — inserts chunks into Supabase's `chunks` table with `source="connector"`

Old chunks are deleted only after new ones are successfully inserted — if embedding fails mid-indexing, existing chunks are preserved so the assistant keeps working.

### 5. User disconnects Zendesk

When a user disconnects from the dashboard, answerhq's `DELETE /api/connectors/connections/{org}/{provider}` does two things:
1. Calls sync_hq to delete the connection (which drops the BYOP schema and synced data)
2. Deletes all connector chunks for that assistant from pgvector

This prevents orphaned chunks from being searched by the AI assistant after the data source is gone.

### 6. AI assistant searches the chunks

When a user asks the assistant a question, `search_chunks()` runs a cosine similarity search across ALL chunk sources (website, article, connector). Connector chunks are automatically included — no special handling needed.

## Environment Variables (answerhq backend)

These must be set in both `.env.local` (local dev) and **Render** (production backend, `answerhq` service):

| Variable | Description | Example |
|----------|-------------|---------|
| `SYNC_HQ_API_URL` | sync_hq base URL | `https://api.synchq.co` |
| `SYNC_HQ_API_KEY` | API key for authenticating to sync_hq | `sk_live_...` |
| `SYNC_HQ_DB_URL` | BYOP Postgres connection string (for reading synced data directly) | `postgresql://sync_hq_writer...` |
| `SYNC_HQ_DEV_ID` | Developer ID (for schema name construction: `sync__{dev_prefix}__{org_id}`) | `171e499f-...` |

## Key Files

| Repo | File | What it does |
|------|------|-------------|
| sync_hq | `sync_hq/sync/engine.py` | Runs the sync and fires the webhook |
| sync_hq | `sync_hq/services/svix_client.py` | Delivers webhooks via Svix |
| answerhq | `api/connectors/index.py` | Receives the webhook, dispatches indexing |
| answerhq | `api/connectors/rag.py` | The indexing pipeline (read → format → chunk → embed → store) |
| answerhq | `api/utils/vector_store.py` | Chunk storage and search in pgvector |
| answerhq | `Connectors.tsx` | Dashboard UI showing sync status and article count |

## Adding a New Provider

When a new provider is added to sync_hq (e.g., Intercom), the only change needed in answerhq is a formatter function that converts that provider's content to markdown. See [answerhq's ADDING_A_NEW_CONNECTOR.md](https://github.com/jackyliang/answer-hq/blob/main/chat-ui/docs/ADDING_A_NEW_CONNECTOR.md).
