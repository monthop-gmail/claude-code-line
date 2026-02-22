# claude-code-line

LINE Messaging API bridge for Claude Code. Chat with Claude Code AI coding agent through LINE.

## Architecture

```
LINE → Cloudflare Tunnel → line-bot (Bun) → server (Claude Agent SDK) → Anthropic API
```

3 containers:
- **server** — Claude Code API server (Hono + Claude Agent SDK)
- **line-bot** — LINE webhook handler (Bun + @line/bot-sdk)
- **cloudflared** — Cloudflare tunnel to expose webhook

The bot can work with any compatible server by changing `SERVER_URL` (e.g. claude-code-server, opencode-server).

## Setup

1. Create a LINE Messaging API channel at https://developers.line.biz/console/
2. In the channel settings:
   - Enable "Use webhook"
   - Issue a "Channel access token (long-lived)"
   - Note the "Channel secret"
3. Get an Anthropic API key at https://console.anthropic.com/
4. Copy and edit `.env`:

```bash
cp .env.example .env
# Edit .env with your credentials
```

## Usage

```bash
docker compose up -d --build
```

Get the tunnel URL:
```bash
docker logs cc-line-tunnel 2>&1 | grep -o 'https://[^ ]*'
```

Set the webhook URL in LINE Developer Console: `https://<tunnel-url>/webhook`

Check server health:
```bash
docker exec cc-line-bot curl -s http://server:4096/health
```

## Commands

- Send any text message to start coding
- `/new` - Start a new coding session
- `/abort` - Cancel the current prompt
- `/sessions` - Show active session info
- `/cost` - Show total cost for current session

## How it works

Each LINE user gets their own Claude Code session. Messages are forwarded to the server API via HTTP, and responses are sent back through LINE.

```
User (LINE app)
  ↕  LINE Messaging API webhook
line-bot (Bun, port 3000)
  ↕  HTTP fetch → http://server:4096
server (Hono + Claude Agent SDK, port 4096)
  ↕  query() → Claude Code CLI (bundled)
Project Filesystem (/workspace)
```

## Environment Variables

### Bot
| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `LINE_CHANNEL_ACCESS_TOKEN` | Yes | - | LINE channel access token |
| `LINE_CHANNEL_SECRET` | Yes | - | LINE channel secret |
| `SERVER_URL` | No | `http://server:4096` | Server API URL |
| `SERVER_PASSWORD` | No | - | Server auth password |
| `PROMPT_TIMEOUT_MS` | No | `300000` | Timeout per prompt (5 min) |

### Server
| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `ANTHROPIC_API_KEY` | Yes | - | Anthropic API key |
| `API_PASSWORD` | No | - | API auth password |
| `CLAUDE_MODEL` | No | `sonnet` | Claude model to use |
| `CLAUDE_MAX_TURNS` | No | `10` | Max agentic turns per prompt |
| `CLAUDE_MAX_BUDGET_USD` | No | `1.00` | Max spend per prompt |

### Alternative Providers

The server supports alternative providers via env vars:

```bash
# Ollama (local)
ANTHROPIC_AUTH_TOKEN=ollama
ANTHROPIC_BASE_URL=http://host.docker.internal:11434

# OpenRouter
ANTHROPIC_API_KEY=sk-or-...
ANTHROPIC_BASE_URL=https://openrouter.ai/api/v1
```
