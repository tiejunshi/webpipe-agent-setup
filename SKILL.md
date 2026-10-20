---
name: webpipe
description: |
  Install, authenticate, and connect WebPipe (https://webpipe.ai) to an AI agent, giving it four MCP tools: webpipe_scrape (a page as Markdown/HTML/JSON), webpipe_map (discover every URL on a site), webpipe_crawl (crawl a whole site), and webpipe_browser (a persistent headless browser that can click, fill, and screenshot). Use when the user wants WebPipe inside Claude Code, Codex, or another MCP client, or needs current web page data through WebPipe. Make setup autonomous: reuse an existing WEBPIPE_API_KEY, otherwise send the user to the WebPipe dashboard for one, install the MCP server with a verbatim command, and verify with a real scrape before declaring setup complete.
---

# WebPipe

WebPipe turns web pages into agent-ready data. It is exposed to an agent as an MCP server with four tools:

| Tool | What it does | Required input |
| --- | --- | --- |
| `webpipe_scrape` | Fetch a single page into Markdown, HTML, links, metadata, or structured JSON | `url` |
| `webpipe_map` | Discover all URLs on a site (robots.txt -> sitemap -> in-page links) | `url` |
| `webpipe_crawl` | Recursively crawl a whole site; the server polls until the job finishes; results are kept for 24 hours | `url` |
| `webpipe_browser` | A persistent headless browser: create a session, run interactions, then scrape the page | `url` or `session_id` |

Every tool also accepts `api_key` (overrides the environment variable) and `base_url`.

## Autonomous Setup

Treat setup as one state machine: **preflight -> install MCP -> authenticate -> verify**. Do not make the user repeat a step that the agent can complete from the terminal.

### 1. Preflight

1. Detect the host agent. Configure only the agent(s) that are actually installed.
2. Check whether WebPipe is already registered:

```bash
claude mcp list      # Claude Code
codex mcp list       # Codex
```

If `webpipe` is already listed, skip installation and go to Verify.

3. Choose the install track:
   - **Track 1 (recommended): remote HTTP MCP** at `https://mcp.webpipe.ai` - use this for any host that supports remote HTTP (streamable HTTP) MCP servers.
   - **Track 2 (fallback): local Python stdio server** - use this only for a host that can run only local stdio MCP servers.
4. Look for an existing WebPipe API key in the environment variable `WEBPIPE_API_KEY`. Reuse it. Never print, log, or commit it.
5. If no key exists, tell the user to create one at https://webpipe.ai/api-keys.html and set it as `WEBPIPE_API_KEY`. Do not invent a key value.

### 2. Install the WebPipe MCP server

#### Track 1 (recommended): remote HTTP MCP

Claude Code:

```bash
claude mcp add --transport http webpipe https://mcp.webpipe.ai \
  --header "Authorization: Bearer $WEBPIPE_API_KEY"
```

Codex:

```bash
codex mcp add webpipe --url https://mcp.webpipe.ai \
  --bearer-token-env-var WEBPIPE_API_KEY
```

If the host stores MCP servers as JSON, add this entry instead:

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

Clients that support environment-variable expansion (for example Claude Code's `.mcp.json`) substitute `${WEBPIPE_API_KEY}` automatically. For clients that do not, put the literal key in place of `${WEBPIPE_API_KEY}`.

#### Track 2 (fallback): local Python stdio server

Install the dependencies first:

```bash
pip install fastmcp webpipe-sdk
```

Source: https://github.com/tiejunshi/webpipe-mcp

Claude Code:

```bash
claude mcp add webpipe --env WEBPIPE_API_KEY=your-webpipe-api-key \
  -- python /absolute/path/to/webpipe-mcp/server.py
```

Codex:

```bash
codex mcp add webpipe --env WEBPIPE_API_KEY=your-webpipe-api-key \
  -- python /absolute/path/to/webpipe-mcp/server.py
```

Generic JSON form:

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

Replace `/absolute/path/to/webpipe-mcp/server.py` with the real absolute path on this machine, and `your-webpipe-api-key` with the key from the dashboard.

### 3. Authenticate

WebPipe uses a single API key. Precedence, highest first:

1. The explicit `api_key` argument passed to a tool call.
2. The environment variable `WEBPIPE_API_KEY`.

How the key travels depends on the track:

- **Track 1 (remote):** the client sends it as the HTTP header `Authorization: Bearer <key>`.
- **Track 2 (stdio):** the client passes it to the server as the environment variable `WEBPIPE_API_KEY`.

Get a key at https://webpipe.ai/api-keys.html. Never print, echo, log, or commit it.

### 4. Verify

Prove the integration works with one real call. Ask the host agent to run:

```text
webpipe_scrape(url="https://example.com")
```

Expected result: a non-error payload whose Markdown contains "Example Domain". If the tool returns `❌ API Key 无效或已过期`, the key is wrong or missing - see Troubleshooting.

One successful scrape is enough to declare Track 1 installed. Optionally exercise `webpipe_map`, `webpipe_crawl`, and `webpipe_browser` on a real site to confirm the full tool surface.

### 5. Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `❌ API Key 无效或已过期` | Key missing, wrong, or expired | Re-check `WEBPIPE_API_KEY`; create a new key at https://webpipe.ai/api-keys.html |
| `⚠️ 超出频率限制（429）` | Rate limit hit | Wait and retry, or upgrade the plan |
| `❌ 参数错误` | Bad tool arguments | Re-check the required `url` and any optional parameters |
| `🌐 网络连接异常（SDK 已自动重试仍失败）` | Outbound network or proxy issue reaching `api.web2json.ai` | Retry after a short wait; verify the machine can reach `api.web2json.ai` |
| `webpipe` not listed / MCP connection failed | Wrong URL or command | Remove and re-add the server; for Track 2 verify the Python path and that `pip install` succeeded |
| Tools not visible after install | Client needs a restart or rescan | Restart the client, then re-run `claude mcp list` / `codex mcp list` |

Only report **WebPipe ready** when all of these are true:

```text
WebPipe MCP server registered in the host agent
API key resolves
A live webpipe_scrape returned real content
```

If the server is registered but the host has not rescanned yet, say **"installed; restart/rescan required"** instead of claiming the tools are active.

## Tool reference

### webpipe_scrape

Fetch one page and return it as Markdown, HTML, links, metadata, or structured JSON.

| Parameter | Type | Notes |
| --- | --- | --- |
| `url` | string | Target page URL (required) |
| `formats` | string[] | Any of `markdown`, `html`, `rawHtml`, `links`, `metadata`, `json`. Default `["markdown"]` |
| `json_schema` | object | Used when `formats` includes `json`; defines the fields to extract. Supports the `x-source` extension |
| `use_proxy` | bool | Enable the backend proxy pool for anti-bot sites |
| `proxy` | string | Custom proxy address (takes precedence over `use_proxy`) |
| `cookie` | string | Cookie string in header format |
| `api_key` / `base_url` | string | Overrides for the API key and API base URL |

### webpipe_map

Discover all URLs on a site and return a link list.

| Parameter | Type | Notes |
| --- | --- | --- |
| `url` | string | Site root URL (required) |
| `limit` | int | Max results, default 5000, max 50000 |
| `search` | string | Keyword filter on link title or description |
| `sitemap` | string | `include` / `only` / `exclude`. Default `include` |
| `same_domain` | bool | Only return same-domain links. Default `true` |
| `use_proxy` / `proxy` / `cookie` / `api_key` / `base_url` | | Same as above |

### webpipe_crawl

Recursively crawl a whole site. The server polls until the crawl finishes, then returns the job; results are kept for 24 hours.

| Parameter | Type | Notes |
| --- | --- | --- |
| `url` | string | Start URL (required) |
| `limit` | int | Max pages, default 100, max 10000 |
| `poll_interval` | float | Polling interval in seconds, default 2.0 |
| `timeout` | float | Overall timeout in seconds; unlimited by default |
| `max_discovery_depth` | int | URL discovery depth |
| `include_paths` / `exclude_paths` | string[] | Regex arrays: only crawl / skip matching paths |
| `allow_subdomains` | bool | Default `false` |
| `allow_external_links` | bool | Default `false` |
| `crawl_entire_domain` | bool | Crawl the entire domain |
| `ignore_query_params` | bool | Ignore URL query parameters |
| `sitemap` | string | `include` / `skip` / `only` |
| `delay` | int | Delay per request in milliseconds, default 0 |
| `max_concurrency` | int | 1-10, default 3 |
| `scrape_formats` | string[] | Formats scraped per page, default `["markdown"]` |
| `use_proxy` / `proxy` / `cookie` / `api_key` / `base_url` | | Same as above |

### webpipe_browser

A persistent headless browser. Three usage patterns:

- Omit `session_id`: create a session, run `actions`, auto-close, and return the result.
- Pass `session_id`: continue operating on an existing session.
- Set `auto_close=false`: keep the session open and return its `session_id` for reuse.

| Parameter | Type | Notes |
| --- | --- | --- |
| `url` | string | Start URL when creating a session (`session_id` empty) |
| `session_id` | string | Existing session ID; skips creation |
| `actions` | object[] | Interaction list, see the table below |
| `navigate_url` | string | Navigate to this URL and scrape it |
| `formats` | string[] | Output formats for the final scrape, default `["markdown"]` |
| `scroll_to_bottom` | bool | Scroll to the bottom before scraping |
| `max_scrolls` | int | Max scrolls (<= 30) |
| `scroll_wait` | int | Wait in milliseconds after each scroll |
| `scroll_step` | int | Pixels per scroll |
| `cookie` | string | Cookie injected when creating the session |
| `proxy` | string | Custom proxy when creating the session |
| `session_name` | string | Persistence name for reusing a logged-in session |
| `auto_close` | bool | Close the session when done, default `true` |
| `api_key` / `base_url` | | Same as above |

`actions[].type` values:

| type | Parameters |
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

## Escalation pattern

Match the tool to the job:

1. **`webpipe_scrape`** when the URL is already known.
2. **`webpipe_map`** when the site is known but the right pages are not.
3. **`webpipe_crawl`** when many pages from a site or section are needed.
4. **`webpipe_browser`** when the page needs JavaScript interaction (login, clicks, form fills, screenshots).
