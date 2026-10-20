---
name: webpipe
description: |
  为 AI 智能体安装、认证并接入 WebPipe（https://webpipe.ai），赋予它 4 个 MCP 工具：webpipe_scrape（把网页转成 Markdown/HTML/JSON）、webpipe_map（发现站点上的全部 URL）、webpipe_crawl（爬取整站）、webpipe_browser（可持续交互的无头浏览器，支持点击、填表、截图）。当用户希望在 Claude Code、Codex 或其他 MCP 客户端内部使用 WebPipe，或需要通过 WebPipe 获取当前网页数据时使用本技能。让安装过程自主完成：优先复用已有的 WEBPIPE_API_KEY，否则引导用户去 WebPipe 控制台获取；用逐字可复制的命令安装 MCP 服务器，并在宣布安装完成前用一次真实抓取做验证。
---

# WebPipe

WebPipe 把网页转成智能体可直接使用的数据。它以 MCP 服务器的形式向智能体暴露 4 个工具：

| 工具 | 作用 | 必填入参 |
| --- | --- | --- |
| `webpipe_scrape` | 抓取单个网页，返回 Markdown、HTML、链接、元数据或结构化 JSON | `url` |
| `webpipe_map` | 发现站点上的全部 URL（robots.txt -> sitemap -> 页内链接） | `url` |
| `webpipe_crawl` | 递归爬取整站；服务端会轮询到任务完成；结果保留 24 小时 | `url` |
| `webpipe_browser` | 持久化无头浏览器：创建会话、执行交互、最后抓取页面 | `url` 或 `session_id` |

所有工具都额外接受 `api_key`（覆盖环境变量）与 `base_url`。

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

3. 选择安装轨道：
   - **轨道 1（推荐）：远程 HTTP MCP**，地址 `https://mcp.webpipe.ai`。只要宿主支持远程 HTTP（streamable HTTP）MCP 就用这条。
   - **轨道 2（备选）：本地 Python stdio 服务器**。仅当宿主只能运行本地 stdio MCP 服务器时使用。
4. 先在环境变量 `WEBPIPE_API_KEY` 中查找已有的 WebPipe API Key。找到就复用。绝不要打印、记录日志或提交它。
5. 如果没有 Key，告诉用户去 https://webpipe.ai/api-keys.html 创建一个，并把它设为 `WEBPIPE_API_KEY`。不要自行编造 Key 值。

### 2. 安装 WebPipe MCP 服务器

#### 轨道 1（推荐）：远程 HTTP MCP

Claude Code：

```bash
claude mcp add --transport http webpipe https://mcp.webpipe.ai \
  --header "Authorization: Bearer $WEBPIPE_API_KEY"
```

Codex：

```bash
codex mcp add webpipe --url https://mcp.webpipe.ai \
  --bearer-token-env-var WEBPIPE_API_KEY
```

如果宿主以 JSON 形式存储 MCP 服务器，则改为添加这条配置：

```json
{
  "mcpServers": {
    "webpipe": {
      "type": "http",
      "url": "https://mcp.webpipe.ai",
      "headers": { "Authorization": "Bearer ${WEBPIPE_API_KEY}" }
    }
  }
}
```

支持环境变量展开的客户端（例如 Claude Code 的 `.mcp.json`）会自动替换 `${WEBPIPE_API_KEY}`。不支持的客户端，请把 `${WEBPIPE_API_KEY}` 换成真实的 Key。

#### 轨道 2（备选）：本地 Python stdio 服务器

先安装依赖：

```bash
pip install fastmcp webpipe-sdk
```

源码：https://github.com/tiejunshi/webpipe-mcp

Claude Code：

```bash
claude mcp add webpipe --env WEBPIPE_API_KEY=your-webpipe-api-key \
  -- python /absolute/path/to/webpipe-mcp/server.py
```

Codex：

```bash
codex mcp add webpipe --env WEBPIPE_API_KEY=your-webpipe-api-key \
  -- python /absolute/path/to/webpipe-mcp/server.py
```

通用 JSON 形式：

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

把 `/absolute/path/to/webpipe-mcp/server.py` 换成这台机器上的真实绝对路径，把 `your-webpipe-api-key` 换成从控制台获取的 Key。

### 3. 认证

WebPipe 使用单一 API Key。优先级从高到低：

1. 工具调用时显式传入的 `api_key` 参数。
2. 环境变量 `WEBPIPE_API_KEY`。

Key 的传递方式取决于轨道：

- **轨道 1（远程）：** 客户端把它作为 HTTP 头 `Authorization: Bearer <key>` 发送。
- **轨道 2（stdio）：** 客户端把它作为环境变量 `WEBPIPE_API_KEY` 传给服务器。

在 https://webpipe.ai/api-keys.html 获取 Key。绝不要打印、echo、记录日志或提交它。

### 4. 验证

用一次真实调用证明接入成功。让宿主智能体执行：

```text
webpipe_scrape(url="https://example.com")
```

预期结果：返回一个非错误的结果，其 Markdown 包含 "Example Domain"。如果工具返回 `❌ API Key 无效或已过期`，说明 Key 有误或缺失，见"常见问题"。

一次成功的抓取即可判定轨道 1 安装完成。可选地，再对一个真实站点跑 `webpipe_map`、`webpipe_crawl`、`webpipe_browser`，确认完整的工具面都可用。

### 5. 常见问题

| 现象 | 可能原因 | 处理 |
| --- | --- | --- |
| `❌ API Key 无效或已过期` | Key 缺失、错误或过期 | 复查 `WEBPIPE_API_KEY`；去 https://webpipe.ai/api-keys.html 新建一个 |
| `⚠️ 超出频率限制（429）` | 触发限流 | 稍等后重试，或升级套餐 |
| `❌ 参数错误` | 工具入参有误 | 复查必填的 `url` 以及可选参数 |
| `🌐 网络连接异常（SDK 已自动重试仍失败）` | 到 `api.web2json.ai` 的出网或代理问题 | 稍等后重试；确认本机能连通 `api.web2json.ai` |
| `webpipe` 未列出 / MCP 连接失败 | URL 或命令写错 | 移除后重新添加；走轨道 2 时确认 Python 路径正确且 `pip install` 成功 |
| 安装后看不到工具 | 客户端需要重启或重新扫描 | 重启客户端，再运行 `claude mcp list` / `codex mcp list` |

只有当以下全部为真时，才报告 **WebPipe ready（WebPipe 已就绪）**：

```text
WebPipe MCP 服务器已在宿主智能体中注册
API Key 可解析
一次真实的 webpipe_scrape 返回了真实内容
```

如果服务器已注册但宿主尚未重新扫描，请说 **"installed; restart/rescan required（已安装；需要重启/重新扫描）"**，而不是声称工具已经活跃。

## 工具参考

### webpipe_scrape

抓取单个网页，返回 Markdown、HTML、链接、元数据或结构化 JSON。

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `url` | string | 目标网页 URL（必填） |
| `formats` | string[] | 可选值：`markdown`、`html`、`rawHtml`、`links`、`metadata`、`json`。默认 `["markdown"]` |
| `json_schema` | object | 当 `formats` 含 `json` 时生效，定义要抽取的字段，支持 `x-source` 扩展 |
| `use_proxy` | bool | 启用后端代理池（反爬场景） |
| `proxy` | string | 自定义代理地址（优先级高于 `use_proxy`） |
| `cookie` | string | Cookie 字符串（Header 格式） |
| `api_key` / `base_url` | string | 覆盖 API Key 与 API 基地址 |

### webpipe_map

发现站点上的全部 URL，返回链接列表。

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `url` | string | 站点根 URL（必填） |
| `limit` | int | 返回数量上限，默认 5000，最大 50000 |
| `search` | string | 关键字过滤（只返回标题或描述包含该词的链接） |
| `sitemap` | string | `include` / `only` / `exclude`，默认 `include` |
| `same_domain` | bool | 是否只返回同域名链接，默认 `true` |
| `use_proxy` / `proxy` / `cookie` / `api_key` / `base_url` | | 同上 |

### webpipe_crawl

递归爬取整站。服务端会轮询到爬取完成再返回任务结果；结果保留 24 小时。

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `url` | string | 起始 URL（必填） |
| `limit` | int | 页面数量上限，默认 100，最大 10000 |
| `poll_interval` | float | 轮询间隔秒数，默认 2.0 |
| `timeout` | float | 总超时秒数，默认不限制 |
| `max_discovery_depth` | int | URL 发现深度 |
| `include_paths` / `exclude_paths` | string[] | 正则数组，只爬 / 跳过匹配的路径 |
| `allow_subdomains` | bool | 默认 `false` |
| `allow_external_links` | bool | 默认 `false` |
| `crawl_entire_domain` | bool | 是否爬整个域名 |
| `ignore_query_params` | bool | 是否忽略 URL 查询参数 |
| `sitemap` | string | `include` / `skip` / `only` |
| `delay` | int | 每次请求延迟毫秒数，默认 0 |
| `max_concurrency` | int | 最大并发数 1-10，默认 3 |
| `scrape_formats` | string[] | 每页抓取的格式列表，默认 `["markdown"]` |
| `use_proxy` / `proxy` / `cookie` / `api_key` / `base_url` | | 同上 |

### webpipe_browser

持久化无头浏览器。三种用法：

- 不传 `session_id`：自动创建会话 -> 执行 `actions` -> 自动关闭 -> 返回结果。
- 传 `session_id`：在已有会话上继续操作。
- `auto_close=false`：保持会话不关闭，返回 `session_id` 供后续复用。

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `url` | string | 创建会话时的起始 URL（`session_id` 为空时生效） |
| `session_id` | string | 已有会话 ID，传入则跳过创建 |
| `actions` | object[] | 交互操作列表，见下表 |
| `navigate_url` | string | 跳转到该 URL 并抓取 |
| `formats` | string[] | 最终 scrape 的输出格式，默认 `["markdown"]` |
| `scroll_to_bottom` | bool | 抓取前是否滚动到底部 |
| `max_scrolls` | int | 最大滚动次数（<= 30） |
| `scroll_wait` | int | 每次滚动等待毫秒数 |
| `scroll_step` | int | 每次滚动像素 |
| `cookie` | string | 创建会话时注入的 Cookie |
| `proxy` | string | 创建会话时的自定义代理 |
| `session_name` | string | 会话持久化名称（用于复用登录态） |
| `auto_close` | bool | 完成后是否自动关闭会话，默认 `true` |
| `api_key` / `base_url` | | 同上 |

`actions[].type` 支持的取值：

| type | 参数 |
| --- | --- |
| `click` | `selector`, `wait_after`? |
| `fill` | `selector`, `value` |
| `select` | `selector`, `value` |
| `press` | `key`, `selector`?, `wait_after`? |
| `scroll` | `direction`?, `amount`? |
| `wait` | `ms` |
| `wait_for` | `selector`, `timeout`? |
| `hover` | `selector` |
| `check` | `selector` |
| `screenshot` | `full_page`? |
| `evaluate` | `expression` |

## 升级模式

按任务选择合适的工具：

1. **`webpipe_scrape`**：URL 已经明确时。
2. **`webpipe_map`**：站点已知，但不确定该用哪些页面时。
3. **`webpipe_crawl`**：需要某站点或区段中多个页面的内容时。
4. **`webpipe_browser`**：页面需要 JavaScript 交互（登录、点击、填表、截图）时。
