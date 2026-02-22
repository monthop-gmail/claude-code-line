# CLAUDE.md

## Project Overview

Claude Code LINE Bot — LINE Messaging API bridge for Claude Code. 3-container architecture: server (Claude Agent SDK) + LINE bot (HTTP client) + Cloudflare tunnel.

## Commands

```bash
# Docker deployment
docker compose up -d --build
docker logs cc-line-bot --tail 30
docker logs cc-server --tail 30

# Check server health
docker exec cc-line-bot curl -s http://server:4096/health

# Get tunnel URL
docker logs cc-line-tunnel 2>&1 | grep -o 'https://[^ ]*'
```

## Architecture

```
User (LINE app)
  ↕  LINE Messaging API webhook
line-bot (Bun, port 3000) — src/index.ts
  ↕  HTTP fetch → http://server:4096
server (Bun + Hono, port 4096) — server/src/
  ↕  Claude Agent SDK query()
Claude Code CLI (bundled in SDK)
  ↕
Project Filesystem (/workspace)
```

### 3 Containers

| Container | Image | Role |
|-----------|-------|------|
| `cc-server` | `./server` (node:22-slim + bun + SDK) | Claude Code API server |
| `cc-line-bot` | `.` (oven/bun:1 + @line/bot-sdk) | LINE webhook handler |
| `cc-line-tunnel` | cloudflare/cloudflared | Expose bot to internet |

## Key Files

- **`src/index.ts`** — LINE bot: webhook, signature validation, message chunking, commands, HTTP client to server
- **`server/src/index.ts`** — Hono API server (routes, SSE, auth)
- **`server/src/claude.ts`** — Claude Agent SDK wrapper (query() AsyncGenerator)
- **`server/src/session.ts`** — In-memory session manager
- **`server/src/events.ts`** — Event bus for SSE

## Key Design

- **Swappable server**: Bot calls server via `SERVER_URL` env — can point to claude-code-server, opencode-server, or any compatible API
- **Server API**: `POST /session` (create), `POST /session/:id/message` (prompt), `POST /session/:id/abort` (cancel), `DELETE /session/:id` (cleanup)
- **Per-user sessions**: Each LINE user gets their own server session
- **Per-user queue**: Messages from same user are serialized (no concurrent prompts)
- **Message chunking**: LINE has 5000 char limit — long responses are split with code block balancing

## LINE Bot Commands

- Any text → forwarded to Claude Code
- `/new` — Clear session, start fresh
- `/abort` — Cancel active prompt
- `/sessions` — Show session info + cost
- `/cost` — Show total cost

## Environment Variables

### Bot (line-bot container)
| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `LINE_CHANNEL_ACCESS_TOKEN` | Yes | - | LINE channel access token |
| `LINE_CHANNEL_SECRET` | Yes | - | LINE channel secret |
| `SERVER_URL` | No | `http://server:4096` | Server API URL |
| `SERVER_PASSWORD` | No | - | Server auth (Bearer token) |
| `PROMPT_TIMEOUT_MS` | No | `300000` | Timeout per prompt (5 min) |

### Server (server container)
| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `ANTHROPIC_API_KEY` | Yes* | - | Anthropic API key |
| `API_PASSWORD` | No | - | API auth password |
| `CLAUDE_MODEL` | No | `sonnet` | Claude model |
| `CLAUDE_MAX_TURNS` | No | `10` | Max agentic turns |
| `CLAUDE_MAX_BUDGET_USD` | No | `1.00` | Max spend per prompt |

## Gotchas

- Tunnel URL changes on restart — update LINE webhook URL
- Bot talks to server via Docker internal DNS (`http://server:4096`)
- Server needs non-root user for Claude Code bypassPermissions mode
- `API_PASSWORD` env is shared between bot (`SERVER_PASSWORD`) and server (`API_PASSWORD`)
