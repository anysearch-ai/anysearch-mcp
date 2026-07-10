# AnySearch MCP Server

Unified real-time search MCP server supporting general web search, vertical domain search, parallel batch search, and full-page URL content extraction.

## Features

- **General Web Search** — open-ended natural language queries
- **Vertical Domain Search** — structured queries across finance, academic, security, legal, code, and more
- **Parallel Batch Search** — execute multiple independent queries in one call
- **URL Content Extraction** — fetch and extract full page content as Markdown
- **Anonymous Access** — works without an API key (with lower rate limits)

## API Key Configuration

An API key is **optional but recommended**. Without a key, all features still work via anonymous access with lower rate limits.

### Register for an API Key (Recommended)

The agent can register the user and obtain an API key in a **single call** — no verification code, no manual signup. Ask the user for a **real email address**: it becomes the account username, and a randomly generated password is emailed to it.

```bash
curl -s -X POST "https://api.anysearch.com/v1/auth/email/register" \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com"}'
```

Success response (`code: 0`) returns the account info and a one-time plaintext API key:

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "username": "you@example.com",
    "email": "you@example.com",
    "login_url": "https://www.anysearch.com/login",
    "api_key": {
      "id": "key_xxxxxxxx",
      "key": "as_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      "key_prefix": "as_sk_xxxxxx...",
      "name": "default",
      "rate_limit": 100,
      "quota_limit": 0,
      "expires_at": null,
      "created_at": "2026-06-23T10:23:00Z"
    }
  }
}
```

On success the agent MUST:

1. Put `data.api_key.key` into the MCP server config as the `Authorization: Bearer <key>` value (and/or set `ANYSEARCH_API_KEY`) — it is shown only once (it can also be retrieved later from the dashboard).
2. Tell the user their username (= email), the `login_url`, and that a **random password has been emailed to that address**.
3. Relay this note to the user: *A verification email has been sent to your inbox. If you don't see it within a few minutes, please check your spam or junk folder. You may need to mark it as "Not Spam" to ensure future emails arrive correctly.*

Error handling (always `code: -1` on error; branch on the `message` string):

| message                           | what to do                                                                                          |
| --------------------------------- | --------------------------------------------------------------------------------------------------- |
| `Invalid email address.`          | ask the user to re-enter the email                                                                  |
| `email_already_registered`        | email is taken — tell the user to sign in at `login_url`; do **not** retry                          |
| contains `Rate limited`           | read retry seconds from message (e.g. `"Rate limited, retry after 300 seconds."`), wait, then retry |
| starts with `Key creation failed.` | account created but key failed — extract the email and URL from the message (`"Key creation failed. Your account <email> was created; sign in at <url>."`) and tell the user to sign in there to create a key manually |
| `Internal server error.`          | retry later or fall back to anonymous                                                               |

> The email **must be real and reachable** — the password is delivered there. There is **no verification code** in this flow; the agent only ever asks for an email.

### Get an API Key

Alternatively, visit https://anysearch.com/console/api-keys to create a free API key.

### Key Priority

| Priority | Source |
|----------|--------|
| 1 (highest) | `--api_key` CLI flag / `Authorization` header |
| 2 | Environment variable `ANYSEARCH_API_KEY` |
| 3 | `.env` file (`ANYSEARCH_API_KEY=<key>`) |
| 4 | Anonymous access (lower rate limits) |

### Key Behavior

| Scenario | Behavior |
|----------|----------|
| No key | Proceed with anonymous access (lower rate limits) |
| Has key | Sent via `Authorization: Bearer <key>` header, higher rate limits |
| Key exhausted, auto-registered key returned | Agent should ask user for confirmation, then persist the new key |
| Key exhausted, no new key | Inform user and suggest configuring a new API key |

## MCP Transport

AnySearch MCP server **natively supports Streamable HTTP** transport (MCP spec 2025-03-26). SSE and stdio clients can connect via proxy.

| Transport | Native? | Best for |
|-----------|---------|----------|
| **Streamable HTTP** | Yes | OpenCode, Claude Desktop (2025.6+), web-based clients |
| **SSE** | Via proxy | Cursor, Windsurf |
| **stdio** | Via proxy | Claude Desktop (legacy), VS Code Copilot, Cline |

## Installation

### Streamable HTTP (Recommended — No Proxy Needed)

For agents that support the Streamable HTTP transport (MCP spec 2025-03-26+):

**OpenCode** (v1.x+ / v0.1.x+):

Config file location depends on your OpenCode version. Run `opencode -v` to check.

| Version | Global Config Path | Project Config Path |
|---------|-------------------|-------------------|
| **1.x+** (current) | `~/.config/opencode/opencode.json` | `opencode.json` or `.opencode/opencode.json` |
| **0.1.x ~ 0.15.x** | `~/.config/opencode/opencode.json` | `opencode.json` |
| **0.0.x** (legacy Go) | `~/.opencode.json` | `.opencode.json` |

> **Windows**: Replace `~/.config/opencode/` with `%USERPROFILE%\.config\opencode\`.

For v1.x+ and v0.1.x+ (MCP key: `mcp`):

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "anysearch": {
      "type": "remote",
      "url": "https://api.anysearch.com/mcp",
      "enabled": true,
      "headers": {
        "Authorization": "Bearer ${ANYSEARCH_API_KEY}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

<details>
<summary>Legacy Go version (0.0.x) — MCP key: <code>mcpServers</code></summary>

```json
{
  "mcpServers": {
    "anysearch": {
      "type": "sse",
      "url": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer ${ANYSEARCH_API_KEY}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

> The legacy Go version does not support Streamable HTTP natively. Use SSE or stdio via proxy instead.

</details>

**Claude Desktop** (2025.6+, `claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "anysearch": {
      "type": "streamable-http",
      "url": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer ${ANYSEARCH_API_KEY}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

> Without an API key, drop only the `Authorization` line but **keep** `X-Anysearch-Client`. The server will use anonymous access automatically.

### stdio (Via Proxy)

For agents that only support stdio transport. Two proxy options:

**Autohand Code** can register the recommended `mcp-remote` proxy while preserving AnySearch's required client header:

```bash
autohand mcp add anysearch -- npx -y mcp-remote https://api.anysearch.com/mcp --header "X-Anysearch-Client: mcp/1.0.0"
```

This uses anonymous access with lower rate limits. Add `--scope project` after `mcp add` to keep the registration in the current workspace. See [Autohand Code](https://github.com/autohandai/code-cli/) for current CLI details.

#### Option A: mcp-remote (Recommended)

[`mcp-remote`](https://github.com/geelen/mcp-remote) — auto-detects Streamable HTTP, simplest config:

**Claude Desktop** (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "anysearch": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://api.anysearch.com/mcp",
        "--header",
        "X-Anysearch-Client: mcp/1.0.0",
        "--header",
        "Authorization: Bearer ${ANYSEARCH_API_KEY}"
      ]
    }
  }
}
```

**VS Code Copilot** (`.vscode/mcp.json`):

```json
{
  "servers": {
    "anysearch": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://api.anysearch.com/mcp",
        "--header",
        "X-Anysearch-Client: mcp/1.0.0",
        "--header",
        "Authorization: Bearer ${ANYSEARCH_API_KEY}"
      ]
    }
  }
}
```

**Cline** (VS Code settings):

```json
{
  "mcpServers": {
    "anysearch": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://api.anysearch.com/mcp",
        "--header",
        "X-Anysearch-Client: mcp/1.0.0",
        "--header",
        "Authorization: Bearer ${ANYSEARCH_API_KEY}"
      ]
    }
  }
}
```

> Without an API key, omit only the `"Authorization: Bearer ..."` `--header` pair; **keep** the `X-Anysearch-Client` `--header`.

#### Option B: supergateway

[`supergateway`](https://github.com/supercorp-ai/supergateway) — more transport options, supports SSE output:

**Claude Desktop** (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "anysearch": {
      "command": "npx",
      "args": [
        "-y",
        "supergateway",
        "--streamableHttp",
        "https://api.anysearch.com/mcp",
        "--header",
        "X-Anysearch-Client: mcp/1.0.0",
        "--oauth2Bearer",
        "${ANYSEARCH_API_KEY}"
      ]
    }
  }
}
```

> Without an API key, omit the `"--oauth2Bearer"` and key args.

### SSE (Via Proxy)

For agents that only support SSE transport (Cursor, Windsurf). Requires running a local SSE proxy server:

#### Start the proxy

```bash
npx -y supergateway \
  --streamableHttp https://api.anysearch.com/mcp \
  --outputTransport sse \
  --port 8000 \
  --header "X-Anysearch-Client: mcp/1.0.0" \
  --oauth2Bearer <your_api_key>
```

> Without an API key, omit the `--oauth2Bearer` flag.

Then configure your agent:

**Cursor** (`.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "anysearch": {
      "type": "sse",
      "url": "http://localhost:8000/sse"
    }
  }
}
```

**Windsurf** (`~/.codeium/windsurf/mcp_config.json`):

```json
{
  "mcpServers": {
    "anysearch": {
      "serverUrl": "http://localhost:8000/sse"
    }
  }
}
```

> The SSE proxy must remain running while the agent is active. Consider running it as a background service.

## Agent Quick Reference

| Agent | Transport | Config Location | Needs Proxy? | Proxy Tool |
|-------|-----------|----------------|-------------|------------|
| OpenCode (v1.x+) | Streamable HTTP | `~/.config/opencode/opencode.json` or project `opencode.json` | No | — |
| Claude Desktop (2025.6+) | Streamable HTTP | `claude_desktop_config.json` | No | — |
| Claude Desktop (legacy) | stdio | `claude_desktop_config.json` | Yes | `mcp-remote` |
| Cursor | SSE | `.cursor/mcp.json` | Yes | `supergateway` |
| VS Code Copilot | stdio | `.vscode/mcp.json` | Yes | `mcp-remote` |
| Windsurf | SSE | `mcp_config.json` | Yes | `supergateway` |
| Cline | stdio | VS Code settings | Yes | `mcp-remote` |

## Available Tools

### `search`

Execute a search query — general or vertical domain.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | Yes | Natural language search query. ONE intent per call |
| `domain` | string | No | Vertical domain (e.g. `finance`, `academic`, `security`). Must come from `get_sub_domains` enum |
| `sub_domain` | string | No | Sub-domain routing key (e.g. `finance.us_stock`). Must come from `get_sub_domains` output |
| `sub_domain_params` | object | No | Structured params from `get_sub_domains` params column. NEVER invent values |
| `max_results` | integer | No | 1–10, default 10 |

### `get_sub_domains`

Query the vertical domain directory. **Required before any search that uses a domain** — returns valid sub_domains and their parameter schemas.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `domain` | string | One of | Single domain to query |
| `domains` | string[] | One of | Batch up to 5 domains (preferred — covers more ground) |

Returns a Markdown table: `sub_domain | description | params`

### `batch_search`

Execute 1–5 independent search queries in parallel. Single failure does not block others.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `queries` | object[] | Yes | 1–5 query objects, each with same fields as `search` |

### `extract`

Fetch full page content from a URL and return as Markdown. Truncated at 50,000 characters. HTML pages only.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `url` | string | Yes | Target URL (`http://` or `https://`) |
