# AnySearch MCP Server

> [English](./README.md) | 简体中文

统一实时搜索 MCP 服务器，支持通用网页搜索、垂直领域搜索、并行批量搜索和整页 URL 内容提取。

## 功能

- **通用网页搜索** —— 开放式自然语言查询
- **垂直领域搜索** —— 支持金融、学术、安全、法律、代码等领域的结构化查询
- **并行批量搜索** —— 在一次调用中执行多个独立查询
- **URL 内容提取** —— 获取并将完整页面内容提取为 Markdown
- **匿名访问** —— 无需 API key 即可使用（速率限制较低）

## API Key 配置

API key **是可选项，但建议配置**。即使没有 key，所有功能仍可通过匿名访问使用，但速率限制较低。

### 注册获取 API Key（推荐）

智能体可以在**一次调用**中完成用户注册并获取 API key —— 无需验证码，无需手动注册。向用户索取一个**真实邮箱地址**：它将作为账户用户名。

```bash
curl -s -X POST "https://api.anysearch.com/v1/auth/email/register" \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com"}'
```

成功响应（`code: 0`）会返回账户信息和仅显示一次的明文 API key：

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

`rate_limit` 等额度字段以实际响应为准，不要把示例值视为固定套餐。

成功后，智能体**必须**：

1. 将 `data.api_key.key` 写入 MCP 服务器配置，作为 `Authorization: Bearer <key>` 的值（和/或设置 `ANYSEARCH_API_KEY`）—— 它只显示一次（之后也可从控制台重新获取）。
2. 告知用户其用户名（= 邮箱）和 `login_url`。

错误响应可能使用 HTTP 状态码、应用级 `code` 或 `message`；不要假设所有错误的 `code` 都是 `-1`：

| 状态 / code / message | 处理方式 |
| --------------------- | -------- |
| `Invalid email address.` | 请用户重新输入邮箱 |
| HTTP `409` 或 `email_already_registered` | 邮箱已被注册 —— 告知用户在 `login_url` 登录；**不要**重试 |
| `code: 42901`、`rate_limited` 或包含 `Rate limited` | 优先读取 `Retry-After`、响应数据或 message 中的重试时间；未提供时间时停止连续请求，稍后再试或先使用匿名访问 |
| 以 `Key creation failed.` 开头 | 账户已创建，但 key 创建失败 —— 从 message 中提取邮箱和 URL（`"Key creation failed. Your account <email> was created; sign in at <url>."`），告知用户在该地址登录并手动创建 key |
| `Internal server error.` | 稍后重试，或回退到匿名访问 |

> 该邮箱**必须真实且可接收邮件**。

### 获取 API Key

也可访问 https://anysearch.com/console/api-keys 创建免费 API key。

### Key 优先级

| 优先级 | 来源 |
| -------- | ---- |
| 1（最高） | `--api_key` CLI 标志 / `Authorization` Header |
| 2 | 环境变量 `ANYSEARCH_API_KEY` |
| 3 | `.env` 文件（`ANYSEARCH_API_KEY=<key>`） |
| 4 | 匿名访问（速率限制较低） |

### Key 行为

| 场景 | 行为 |
| ---- | ---- |
| 无 key | 使用匿名访问（速率限制较低） |
| 有 key | 通过 `Authorization: Bearer <key>` Header 发送，享有更高的速率限制 |
| Key 额度耗尽，返回自动注册的 key | 智能体应请求用户确认，然后持久化保存新 key |
| Key 额度耗尽，未返回新 key | 告知用户，并建议配置新 API key |

## MCP 传输

线上服务端点为：

```text
https://api.anysearch.com/mcp
```

该端点原生使用 **Streamable HTTP**。当前版本的 OpenCode、Claude Code、Cursor、VS Code、Cline、OpenAI Codex、Google Antigravity、DeepSeek Harness、Hermes Agent 和 OMP 均可直接连接，无需 SSE 或 stdio 代理。以下配置均依据各客户端当前官方文档整理。

## 安装

API key 是可选项。若客户端官方文档说明了自定义 Header 的环境变量或密钥存储方式，下面默认给出推荐的鉴权配置；若未说明，则示例保持匿名，并说明如何只在私有用户配置中添加鉴权。其他示例如需匿名访问，只删除 `Authorization` 项，并保留 `X-Anysearch-Client`。

### OpenCode

官方文档：[MCP servers](https://opencode.ai/docs/mcp-servers/) 和 [配置文件位置](https://opencode.ai/docs/config/)。

全局配置使用 `~/.config/opencode/opencode.json`，项目配置使用项目根目录下的 `opencode.json`。Windows 全局路径为 `%USERPROFILE%\.config\opencode\opencode.json`。

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "anysearch": {
      "type": "remote",
      "url": "https://api.anysearch.com/mcp",
      "enabled": true,
      "oauth": false,
      "headers": {
        "Authorization": "Bearer {env:ANYSEARCH_API_KEY}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

OpenCode 使用 `{env:NAME}` 语法引用环境变量。`"oauth": false` 可避免 API key 鉴权服务触发不必要的 OAuth 探测。

保存后运行 `opencode mcp list`，确认 `anysearch` 显示为 `connected`。

### Claude Code

官方文档：[通过 MCP 将 Claude Code 连接到工具](https://code.claude.com/docs/en/mcp)。

如需仅供当前用户使用、并在本机所有项目中生效，运行：

```bash
claude mcp add --transport http anysearch https://api.anysearch.com/mcp --scope user --header 'Authorization: Bearer ${ANYSEARCH_API_KEY}' --header 'X-Anysearch-Client: mcp/1.0.0'
```

`user` scope 会将服务写入 `~/.claude.json`，并在本机所有项目中提供。单引号会保留变量引用，避免把明文 key 写进配置；启动 Claude Code 前需设置 `ANYSEARCH_API_KEY`。匿名访问时，省略 `Authorization` 对应的 `--header` 参数。

如需共享项目配置，在项目根目录创建 `.mcp.json`：

```json
{
  "mcpServers": {
    "anysearch": {
      "type": "http",
      "url": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer ${ANYSEARCH_API_KEY}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

Claude Code 会在 `.mcp.json` 中展开 `${VAR}` 和 `${VAR:-default}`，包括 `url` 与 `headers`。启动 Claude Code 前需设置 `ANYSEARCH_API_KEY`。首次以交互方式打开项目级 MCP 服务时需要审批。

使用以下命令验证连接：

```bash
claude mcp get anysearch
claude mcp list
```

在 Claude Code 内运行 `/mcp` 可查看服务状态和可用工具。

### Cursor

官方文档：[Model Context Protocol](https://cursor.com/docs/mcp)。

项目配置使用 `.cursor/mcp.json`，全局配置使用 `~/.cursor/mcp.json`：

```json
{
  "mcpServers": {
    "anysearch": {
      "url": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer ${env:ANYSEARCH_API_KEY}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

Cursor 会自动识别远程 HTTP 端点，并支持在 `url` 和 `headers` 中使用 `${env:NAME}` 环境变量插值。

可在 **Customize > MCPs** 或 **Output > MCP Logs** 查看连接；已安装 Cursor CLI 时也可运行：

```bash
agent mcp list
agent mcp list-tools anysearch
```

### VS Code

官方文档：[添加和管理 MCP 服务器](https://code.visualstudio.com/docs/agent-customization/mcp-servers) 和 [MCP 配置参考](https://code.visualstudio.com/docs/agents/reference/mcp-configuration)。

运行 **MCP: Open User Configuration** 可配置全局服务，也可在工作区创建 `.vscode/mcp.json`。下面使用密码输入变量，首次启动时提示输入 key 并安全保存，避免将其提交到仓库：

```json
{
  "inputs": [
    {
      "type": "promptString",
      "id": "anysearch-api-key",
      "description": "AnySearch API key",
      "password": true
    }
  ],
  "servers": {
    "anysearch": {
      "type": "http",
      "url": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer ${input:anysearch-api-key}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

如需匿名访问，删除 `Authorization` 项和整个 `inputs` 数组。

运行 **MCP: List Servers**，选择 `anysearch` 后执行 **Start** 或 **Show Output** 验证连接。注意：`${input:...}` 适用于本机 VS Code，但需要交互输入的服务器不会转发到 Agent Host。需要跨 Agent Host/Copilot 工具复用时，改用工作区 `.mcp.json` 或用户 `~/.copilot/mcp-config.json`，并避免依赖交互式 input。

### Cline

官方文档：[MCP](https://docs.cline.bot/mcp/mcp-overview)。

在 Cline 面板中打开 **MCP Servers > Configure > Configure MCP Servers**，也可从 **Remote Servers** 标签页添加托管端点。必须显式使用 `streamableHttp` 类型；省略时会为兼容旧配置而回退到 SSE。

```json
{
  "mcpServers": {
    "anysearch": {
      "type": "streamableHttp",
      "url": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer ${env:ANYSEARCH_API_KEY}",
        "X-Anysearch-Client": "mcp/1.0.0"
      },
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

Cline IDE 支持在 `url`、`headers` 和 `env` 中使用 `${env:VAR}`。运行 `cline mcp` 可打开 CLI 交互式向导，`cline config mcp --json` 可列出配置。官方网页仍写 `~/.cline/mcp.json`，但当前源码默认使用 `~/.cline/data/settings/cline_mcp_settings.json`；因此应通过 IDE 的 **Configure MCP Servers** 或 CLI 向导打开实际配置，不要依赖硬编码路径。

以下五项已于 2026-08-27 对照一手配置文档核查。该结论表示客户端文档级兼容，不等同于已完成本机端到端联通认证。

### OpenAI Codex

官方文档：[Model Context Protocol](https://developers.openai.com/codex/mcp/)。

全局配置使用 `~/.codex/config.toml`，受信任项目可使用 `.codex/config.toml`。Codex CLI、IDE 扩展和桌面应用共享该配置：

```toml
[mcp_servers.anysearch]
url = "https://api.anysearch.com/mcp"
bearer_token_env_var = "ANYSEARCH_API_KEY"
http_headers = { "X-Anysearch-Client" = "mcp/1.0.0" }
```

启动 Codex 前需设置 `ANYSEARCH_API_KEY`。匿名访问时删除 `bearer_token_env_var`。运行 `codex mcp list`，或在 Codex TUI 中运行 `/mcp`，确认服务及工具可用。

### Google Antigravity

官方文档：[Antigravity 中的 MCP 服务](https://antigravity.google/docs/ide/mcp)。

全局配置使用 `~/.gemini/config/mcp_config.json`，工作区配置使用 `.agents/mcp_config.json`。远程服务必须使用 `serverUrl`；不支持旧的 `url` 和 `httpUrl` 键：

```json
{
  "mcpServers": {
    "anysearch": {
      "serverUrl": "https://api.anysearch.com/mcp",
      "headers": {
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

Antigravity 官方文档目前只展示 Header 明文值，未说明远程 Header 的环境变量插值方式。如需鉴权，仅在私有全局配置中添加 `"Authorization": "Bearer YOUR_API_KEY"`，绝不要提交到 `.agents/mcp_config.json`。在 IDE 中打开 **MCP Servers > Manage MCP Servers**，或在 CLI 中运行 `/mcp`，可查看实时状态、日志和工具。

### DeepSeek Harness

官方资料：[DeepSeek Harness](https://www.deepseek.com/harness/en/) 和 [`@deepseek-ai/dsh-mcp-client`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-client/README.md)。

DeepSeek Harness 通过官方 Cordis 插件配置 MCP，而不是使用独立的 `mcp.json`。将下面的 patch 合并到 `~/.dsh/profiles/web/cordis.patch.yml`（或 `$DSH_HOME` 下对应 profile）：

```yaml
- insert:
    - id: mcp-anysearch
      name: "@deepseek-ai/dsh-mcp-client"
      config:
        serverName: anysearch
        transport: streamable-http
        url: "https://api.anysearch.com/mcp"
        headers:
          Authorization: !!js '`Bearer ${process.env.ANYSEARCH_API_KEY}`'
          X-Anysearch-Client: "mcp/1.0.0"
        failOnStartupError: true
```

设置 `ANYSEARCH_API_KEY` 后，先运行 `npx @deepseek-ai/dsh --profile web --dump-config`，确认 profile 已包含 `mcp-anysearch`，再运行 `npx @deepseek-ai/dsh web`。当前 MCP 插件只暴露 tools，尚不支持 MCP resources 和 prompts。DeepSeek Harness 仍处于 developer preview，升级时应重新核对上述插件文档。

### Hermes Agent

官方文档：[MCP 集成](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/mcp.md)。

将服务添加到 `$HERMES_HOME/config.yaml`，并把 `ANYSEARCH_API_KEY` 放入 `$HERMES_HOME/.env` 或进程环境。macOS 和 Linux 的 `$HERMES_HOME` 默认为 `~/.hermes`；原生 Windows 安装器当前默认为 `%LOCALAPPDATA%\hermes`：

```yaml
mcp_servers:
  anysearch:
    url: "https://api.anysearch.com/mcp"
    headers:
      Authorization: "Bearer ${ANYSEARCH_API_KEY}"
      X-Anysearch-Client: "mcp/1.0.0"
```

运行 `hermes mcp test anysearch` 和 `hermes mcp list` 验证连接。如在会话中修改了配置，运行 `/reload-mcp` 或新建会话。

### OMP (Oh My Pi)

官方文档：[MCP 配置](https://github.com/can1357/oh-my-pi/blob/main/docs/mcp-config.md)。

项目配置使用 `.omp/mcp.json`，全局配置使用 `~/.omp/agent/mcp.json`：

```json
{
  "$schema": "https://raw.githubusercontent.com/can1357/oh-my-pi/main/packages/coding-agent/src/config/mcp-schema.json",
  "mcpServers": {
    "anysearch": {
      "type": "http",
      "url": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer ${ANYSEARCH_API_KEY}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

OMP 会在配置值中展开 `${VAR}` 和 `${VAR:-default}`。设置 `ANYSEARCH_API_KEY` 后，在 OMP 中依次运行 `/mcp reload`、`/mcp test anysearch` 和 `/mcp list`。

## 客户端速查表

| 客户端 | 官方接入方式 | 配置位置 | 可直连 Streamable HTTP？ |
| ------ | ------------ | -------- | ------------------------ |
| OpenCode | 远程 MCP 配置 | `~/.config/opencode/opencode.json` 或项目 `opencode.json` | 是 |
| Claude Code | 远程 HTTP MCP | 用户级 `~/.claude.json` 或项目级 `.mcp.json` | 是 |
| Cursor | 远程 MCP 配置 | `.cursor/mcp.json` 或 `~/.cursor/mcp.json` | 是 |
| VS Code | HTTP MCP 配置 | 本机用户配置或 `.vscode/mcp.json`；Agent Host 使用 `.mcp.json` / `~/.copilot/mcp-config.json` | 是 |
| Cline | 远程 Streamable HTTP | 通过 Cline IDE 或 CLI 向导打开实际配置 | 是 |
| OpenAI Codex | 远程 MCP 配置 | `~/.codex/config.toml` 或受信任项目 `.codex/config.toml` | 是 |
| Google Antigravity | 使用 `serverUrl` 的远程 MCP 配置 | `~/.gemini/config/mcp_config.json` 或工作区 `.agents/mcp_config.json` | 是 |
| DeepSeek Harness | 官方 MCP client 插件 | `$DSH_HOME/profiles/<name>/cordis.patch.yml` | 是 |
| Hermes Agent | 远程 HTTP MCP | `$HERMES_HOME/config.yaml` | 是 |
| OMP (Oh My Pi) | HTTP MCP 配置 | `.omp/mcp.json` 或 `~/.omp/agent/mcp.json` | 是 |

## 可用工具

### `search`

执行通用或垂直领域搜索查询。

| 参数 | 类型 | 必填 | 说明 |
| ---- | ---- | ---- | ---- |
| `query` | string | 是 | 自然语言搜索查询。每次调用只包含一个意图 |
| `domain` | string | 否 | 垂直领域（例如 `finance`、`academic`、`security`）。必须来自 `get_sub_domains` 枚举 |
| `sub_domain` | string | 否 | 子领域路由 key（例如 `finance.us_stock`）。必须来自 `get_sub_domains` 输出 |
| `sub_domain_params` | object | 否 | 来自 `get_sub_domains` params 列的结构化参数。**绝不要**臆造参数值 |
| `max_results` | integer | 否 | 1–10，默认为 10 |

### `get_sub_domains`

查询垂直领域目录。**使用 domain 进行任何搜索前都必须先调用** —— 返回有效的 sub_domains 及其参数 schema。

| 参数 | 类型 | 必填 | 说明 |
| ---- | ---- | ---- | ---- |
| `domain` | string | 二选一 | 要查询的单个领域 |
| `domains` | string[] | 二选一 | 批量查询最多 5 个领域（推荐 —— 覆盖范围更广） |

返回 Markdown 表格：`sub_domain | description | params`

### `batch_search`

并行执行 1–5 个独立搜索查询。单个查询失败不会阻塞其他查询。

| 参数 | 类型 | 必填 | 说明 |
| ---- | ---- | ---- | ---- |
| `queries` | object[] | 是 | 1–5 个查询对象，每个对象的字段与 `search` 相同 |

### `extract`

从 URL 获取完整页面内容并以 Markdown 返回。内容超过 50,000 个字符时会被截断。仅支持 HTML 页面。

| 参数 | 类型 | 必填 | 说明 |
| ---- | ---- | ---- | ---- |
| `url` | string | 是 | 目标 URL（`http://` 或 `https://`） |
