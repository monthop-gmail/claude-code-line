# claude-code-line

LINE Messaging API bridge for Claude Code. Chat with Claude Code AI coding agent through LINE.

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
docker logs claude-code-line-tunnel 2>&1 | grep trycloudflare.com
```

Set the webhook URL in LINE Developer Console: `https://<tunnel-url>/webhook`

## Commands

- Send any text message to start coding
- `/new` - Start a new coding session
- `/abort` - Cancel the current prompt
- `/sessions` - Show active session info
- `/cost` - Show total cost for current session

## How it works

Each LINE user gets their own Claude Code session. Messages are forwarded to the Claude Code CLI via subprocess, and responses are sent back through LINE.

```
User (LINE app)
  ↕  LINE Messaging API webhook
claude-code-line (Bun HTTP server)
  ↕  spawn("claude", ["-p", prompt, "--resume", sessionId])
Claude Code CLI
  ↕
Project Filesystem (/workspace)
```

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `LINE_CHANNEL_ACCESS_TOKEN` | Yes | - | LINE channel access token |
| `LINE_CHANNEL_SECRET` | Yes | - | LINE channel secret |
| `ANTHROPIC_API_KEY` | Yes | - | Anthropic API key |
| `CLAUDE_MODEL` | No | `sonnet` | Claude model to use |
| `CLAUDE_MAX_TURNS` | No | `10` | Max agentic turns per prompt |
| `CLAUDE_MAX_BUDGET_USD` | No | `1.00` | Max spend per prompt |
| `CLAUDE_TIMEOUT_MS` | No | `300000` | Timeout per prompt (5 min) |
