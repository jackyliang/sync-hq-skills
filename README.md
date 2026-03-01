# sync-hq-skills

AI agent skill for integrating with [sync_hq](https://sync-hq.com) — sync third-party data (Zendesk, etc.) into Postgres for your AI agents to query.

## Installation

### Claude Code

```bash
# From your project root
mkdir -p .claude/skills/sync-hq
cp SKILL.md .claude/skills/sync-hq/SKILL.md
```

Claude Code will automatically pick up the skill and reference it when you ask about sync_hq integration.

### Cursor

```bash
mkdir -p .cursor/skills
cp SKILL.md .cursor/skills/sync-hq.md
```

### Other AI Agents

Copy `SKILL.md` into wherever your agent reads skill/instruction files. The file is self-contained — no other files needed.

## What's Inside

`SKILL.md` teaches your AI agent how to:

1. **Authenticate** with the sync_hq API
2. **Connect providers** (Zendesk, etc.) for your end users via OAuth
3. **Set up and manage syncs** — trigger, schedule, monitor
4. **Read synced data** from Postgres via the API
5. **Configure webhooks** for real-time sync notifications

## Environment Variables

Your project needs these set for the agent to make API calls:

| Variable | Description |
|----------|-------------|
| `SYNC_HQ_API_URL` | sync_hq base URL (e.g., `https://api.sync-hq.com`) |
| `SYNC_HQ_API_KEY` | Your API key (starts with `sk_live_`) |

## License

MIT
