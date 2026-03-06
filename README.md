# sync-hq

Claude Code plugin for integrating [sync_hq](https://github.com/jackyliang/sync_hq) — sync third-party data (Zendesk, etc.) into Postgres for AI agents to query.

## Install

### From GitHub marketplace (recommended)

```bash
# In Claude Code, add the marketplace:
/plugin marketplace add jackyliang/sync-hq-skills

# Then install the plugin:
/plugin install sync-hq@sync-hq
```

### From a local clone

```bash
git clone https://github.com/jackyliang/sync-hq-skills.git
/plugin marketplace add ./sync-hq-skills
/plugin install sync-hq@sync-hq
```

### For a single session (no install)

```bash
claude --plugin-dir /path/to/sync-hq-skills
```

After installing, the `sync-hq` skill is automatically available. Claude will use it when you ask about sync_hq integration, connecting providers, or reading synced data.

## What's Inside

The plugin teaches Claude how to:

1. **Connect providers** (Zendesk, etc.) for end users via OAuth
2. **Set up and manage syncs** — trigger, schedule, monitor
3. **Configure BYOP** — write synced data to your own Postgres
4. **Query synced data** — via API (hosted) or direct SQL (BYOP)
5. **Set up webhooks** for real-time sync notifications
6. **Debug failures** — check status, read logs, test connections

## Plugin Structure

```
sync-hq-skills/
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest
├── skills/
│   └── sync-hq/
│       ├── SKILL.md          # Overview, workflow, navigation
│       ├── api-reference.md  # Full endpoint reference
│       ├── byop-guide.md     # BYOP setup, schema naming, querying
│       └── recipes.md        # Common recipes and troubleshooting
├── README.md
└── LICENSE
```

## Environment Variables

Your project needs these for the agent to make API calls:

| Variable | Description |
|----------|-------------|
| `SYNC_HQ_API_URL` | sync_hq base URL (e.g., `https://api.sync-hq.com`) |
| `SYNC_HQ_API_KEY` | Your API key (starts with `sk_live_`) |
| `SYNC_HQ_DEV_ID` | Your developer ID (for BYOP schema naming) |

## License

MIT
