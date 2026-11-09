---
name: webpipe
description: |
  为 AI 智能体安装、认证并接入 WebPipe（https://webpipe.ai），赋予它 5 个 MCP 工具：webpipe_verify_auth（确认 API Key 可用）、webpipe_scrape（把网页转成 Markdown/HTML/JSON）、webpipe_map（发现站点上的全部 URL）、webpipe_crawl（爬取整站）、webpipe_browser（可持续交互的无头浏览器，支持点击、填表、截图）。当用户希望在 Claude Code、Codex、Cursor、Claude Desktop、Windsurf、Cline 或其他 MCP 客户端内部使用 WebPipe，或需要通过 WebPipe 获取当前网页数据时使用本技能。让安装过程自主完成：优先复用已有的 WEBPIPE_API_KEY，否则引导用户去 WebPipe 控制台获取；用逐字可复制的命令安装 MCP 服务器，并在宣布安装完成前用 webpipe_verify_auth 加一次真实抓取做验证。
---

# WebPipe

WebPipe 把网页转成智能体可直接使用的数据。它以 MCP 服务器的形式向智能体暴露 5 个工具：

| 工具 | 作用 | 关键入参 |
| --- | --- | --- |
| `webpipe_verify_auth` | 成本最低的一次真实 API 调用；确认 Key 有效 | — |
| `webpipe_scrape` | 抓取单个网页，返回 Markdown、HTML、链接、元数据或结构化 JSON | `url` |
| `webpipe_map` | 发现站点上的全部 URL（robots.txt -> sitemap -> 页内链接） | `url` |
| `webpipe_crawl` | 递归爬取整站；服务端会轮询到任务完成；结果保留 24 小时 | `url` |
| `webpipe_browser` | 持久化无头浏览器：创建会话、执行交互、最后抓取页面 | `url` 或 `session_id` |

所有工具都额外接受 `api_key`（覆盖环境变量）与 `base_url`。

完整参数参考：MCP 客户端里展示的工具 schema、[server.py](https://github.com/tiejunshi/webpipe-mcp) 的 docstring（每个工具都带输出示例）、或该仓库的 README。本文件刻意只保留最必要的部分。

## 自主安装

把安装过程视为一个状态机：**预检 -> 安装 MCP -> 认证 -> 验证**。不要让用户去做任何智能体能从终端完成的步骤。

### 1. 预检

1. 探测当前宿主智能体。只配置实际已安装的那个/那些智能体。
2. 检查 WebPipe 是否已经注册：

```bash
claude mcp list      # Claude Code
codex mcp list       # Codex
```

如果 `webpipe` 已经列出，跳过安装，直接进入"认证 / 验证"。

3. WebPipe 目前只提供本地 Python stdio MCP 服务器这一种形态（没有托管的远程服务器）。下一步安装即覆盖这唯一路径。
4. 先在环境变量 `WEBPIPE_API_KEY` 中查找已有的 WebPipe API Key。找到就复用。绝不要打印、记录日志或提交它。
5. 如果没有 Key，告诉用户去 https://webpipe.ai/api-keys.html 创建一个，并把它设为 `WEBPIPE_API_KEY`。不要自行编造 Key 值。

### 2. 安装 WebPipe MCP 服务器

#### 本地（Python stdio 服务器）

先安装依赖并获取服务器代码：

```bash
pip install fastmcp webpipe-sdk
# 服务器源码：https://github.com/tiejunshi/webpipe-mcp
```

**Claude Code**（CLI）：

```bash
claude mcp add webpipe --env WEBPIPE_API_KEY=your-webpipe-api-key \
  -- python /absolute/path/to/webpipe-mcp/server.py
```

**Codex**（CLI）：

```bash
codex mcp add webpipe --env WEBPIPE_API_KEY=your-webpipe-api-key \
  -- python /absolute/path/to/webpipe-mcp/server.py
```

**JSON 配置类智能体。** 下面的模板以 **Claude Desktop 的配置文件格式**为模版（`mcpServers` + `command`/`args`/`env`——这是大多数客户端沿用的 stdio 事实模版）。其他智能体结构相同但文件位置不同——见模板后的表格：

```json
{
  "mcpServers": {
    "webpipe": {
      "command": "python",
      "args": ["/absolute/path/to/webpipe-mcp/server.py"],
      "env": { "WEBPIPE_API_KEY": "your-webpipe-api-key" }
    }
  }
}
```

| 智能体 | 配置文件 | 说明 |
| --- | --- | --- |
| Claude Code | `.mcp.json`（项目）或 `~/.claude.json` | 自动展开 `${WEBPIPE_API_KEY}` |
| Cursor | `~/.cursor/mcp.json`（全局）或 `.cursor/mcp.json`（项目） | 同样是 `mcpServers` 结构 |
| Windsurf | `~/.codeium/windsurf/mcp_config.json` | 同样是 `mcpServers` 结构 |
| Claude Desktop | Windows：`%APPDATA%\Claude\claude_desktop_config.json`；macOS：`~/Library/Application Support/Claude/claude_desktop_config.json` | 使用相同的 JSON 结构 |
| Cline | 在 Cline 的 "MCP Servers" 面板中配置 | 面板接受相同的 `mcpServers` JSON |
| VS Code（Copilot） | `.vscode/mcp.json` | 结构不同：顶层键是 `"servers"` 而不是 `"mcpServers"` |
| Gemini CLI | `~/.gemini/settings.json` | 设置中的 `mcpServers` 键 |

支持环境变量展开的客户端（例如 Claude Code）会自动替换 `${WEBPIPE_API_KEY}`。不支持的客户端，请把 `${WEBPIPE_API_KEY}` 换成真实的 Key——并提醒用户：此时 Key 会明文存在配置文件里。

把 `/absolute/path/to/webpipe-mcp/server.py` 换成这台机器上的真实绝对路径，把 `your-webpipe-api-key` 换成从控制台获取的 Key。

### 3. 认证

WebPipe 使用单一 API Key。优先级从高到低：

1. 工具调用时显式传入的 `api_key` 参数。
2. 环境变量 `WEBPIPE_API_KEY`。

客户端把 Key 作为环境变量 `WEBPIPE_API_KEY` 传给服务器。

在 https://webpipe.ai/api-keys.html 获取 Key。绝不要打印、echo、记录日志或提交它。

### 4. 验证

用两次真实调用证明接入成功。让宿主智能体执行：

```text
webpipe_verify_auth()
```

预期：返回 `{"authenticated": true, ...}`。如果返回 `❌ API key is invalid or expired`，说明 Key 缺失或错误——先修复 `WEBPIPE_API_KEY` 并重新运行这一步，再做其他事。

再一次真实抓取：

```text
webpipe_scrape(url="https://example.com")
```

预期结果：返回一个非错误的结果，其 Markdown 包含 "Example Domain"。

一次成功的 `webpipe_verify_auth` 即可判定安装完成。可选地，再对一个真实站点跑 `webpipe_map`、`webpipe_crawl`、`webpipe_browser`，确认完整的工具面都可用。

### 5. 常见问题

所有工具都把错误作为可读字符串返回（绝不抛异常），前缀为 ❌ 认证/参数/服务端、⚠️ 限流、🌐 网络、🔴 后端——每条消息都附带推荐的恢复步骤。常见情况：

| 现象 | 可能原因 | 处理 |
| --- | --- | --- |
| `❌ API key is invalid or expired` | Key 缺失、错误或过期 | 修复 `WEBPIPE_API_KEY`；运行 `webpipe_verify_auth` 确认；去 https://webpipe.ai/api-keys.html 新建一个 |
| `⚠️ Rate limit exceeded (429)` | 触发限流 | 稍等后按退避策略重试；爬取时调低 `max_concurrency`/`limit` |
| `❌ Bad request` | 工具入参有误 | 复查必填的 `url` 以及可选参数 |
| `🌐 Network error` | 到 `api.web2json.ai` 的出网或代理问题 | 稍等后重试一次；确认本机能连通 `api.web2json.ai` |
| `❌ Resource not found or expired` | 爬取任务已超过 24 小时 | 重新执行爬取 |
| `webpipe` 未列出 / MCP 连接失败 | URL 或命令写错 | 移除后重新添加；确认 Python 路径正确且 `pip install` 成功 |
| 安装后看不到工具 | 客户端需要重启或重新扫描 | 重启客户端，再运行 `claude mcp list` / `codex mcp list` |

只有当以下全部为真时，才报告 **WebPipe ready（WebPipe 已就绪）**：

```text
WebPipe MCP 服务器已在宿主智能体中注册
webpipe_verify_auth 返回 authenticated: true
一次真实的 webpipe_scrape 返回了真实内容
```

如果服务器已注册但宿主尚未重新扫描，请说 **"installed; restart/rescan required（已安装；需要重启/重新扫描）"**，而不是声称工具已经活跃。

## 工具参考（要点）

完整的输入/输出说明在工具 schema 和 [server.py](https://github.com/tiejunshi/webpipe-mcp) 的 docstring 中（每个工具都带输出示例）。这里只重复与决策相关的要点。

- **webpipe_verify_auth** — 无必填参数。安装验证时以及遇到任何认证错误后，先调用它。
- **webpipe_scrape** — `url` 必填。`formats`：`markdown`（默认）、`html`、`rawHtml`、`links`、`metadata`、`json`；`json_schema` 指导 JSON 字段抽取；`use_proxy`/`proxy`/`cookie` 用于反爬站点。
- **webpipe_map** — `url` 必填。`limit`（默认 5000）、`search` 关键字过滤、`sitemap` include/only/exclude、`same_domain`（默认 true）。
- **webpipe_crawl** — `url` 必填。`limit`（默认 100，最大 10000）、`max_discovery_depth`、`include_paths`/`exclude_paths` 正则、`max_concurrency`（1-10）、`scrape_formats`。阻塞到完成；结果保留 24 小时。
- **webpipe_browser** — 三种用法：
  - 一次性：不传 `session_id` —— 创建会话 -> 执行 `actions` -> 抓取 -> 自动关闭。
  - 持久会话：`auto_close=false` —— 响应中带 `session_id`；下次调用传入它即可延续浏览器状态（登录态、Cookie）。
  - 可复用登录态：`session_name` + `auto_close=false` —— 把登录后的状态以名字保存，之后按名字恢复。
  - `actions` 类型：`click`、`fill`、`select`、`press`、`scroll`、`wait`、`wait_for`、`hover`、`check`、`screenshot`、`evaluate`。

## 升级模式

按任务选择合适的工具：

1. **`webpipe_verify_auth`**：认证状态未确认时。
2. **`webpipe_scrape`**：URL 已经明确时。
3. **`webpipe_map`**：站点已知，但不确定该用哪些页面时。
4. **`webpipe_crawl`**：需要某站点或区段中多个页面的内容时。
5. **`webpipe_browser`**：页面需要 JavaScript 交互（登录、点击、填表、截图）时。
