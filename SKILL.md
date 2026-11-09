---
name: webpipe
description: |
  Install, authenticate, and connect WebPipe (https://webpipe.ai) to an AI agent, giving it five MCP tools: webpipe_verify_auth (confirm the API key works), webpipe_scrape (a page as Markdown/HTML/JSON), webpipe_map (discover every URL on a site), webpipe_crawl (crawl a whole site), and webpipe_browser (a persistent headless browser that can click, fill, and screenshot). Use when the user wants WebPipe inside Claude Code, Codex, Cursor, Claude Desktop, Windsurf, Cline, or another MCP client, or needs current web page data through WebPipe. Make setup autonomous: reuse an existing WEBPIPE_API_KEY, otherwise send the user to the WebPipe dashboard for one, install the MCP server with a verbatim command, and verify with webpipe_verify_auth plus one real scrape before declaring setup complete.
---

# WebPipe

WebPipe turns web pages into agent-ready data. It is exposed to an agent as an MCP server with five tools:

| Tool | What it does | Key inputs |
| --- | --- | --- |
| `webpipe_verify_auth` | Cheapest real API call; confirms the key is valid | — |
| `webpipe_scrape` | Fetch a single page into Markdown, HTML, links, metadata, or structured JSON | `url` |
| `webpipe_map` | Discover all URLs on a site (robots.txt -> sitemap -> in-page links) | `url` |
| `webpipe_crawl` | Recursively crawl a whole site; the server polls until the job finishes; results are kept for 24 hours | `url` |
| `webpipe_browser` | A persistent headless browser: create a session, run interactions, then scrape the page | `url` or `session_id` |

Every tool also accepts `api_key` (overrides the environment variable) and `base_url`.

Full parameter reference: the tool schema shown in your MCP client, the docstrings in [server.py](https://github.com/tiejunshi/webpipe-mcp) (each includes an output example), or the README of that repository. This file intentionally keeps only the essentials.

## Autonomous Setup

Treat setup as one state machine: **preflight -> install MCP -> authorize (client-side) -> authenticate -> verify**. Do not make the user repeat a step that the agent can complete from the terminal.

### 1. Preflight

1. Detect the host agent. Configure only the agent(s) that are actually installed.
2. Check whether WebPipe is already registered:

```bash
claude mcp list      # Claude Code
codex mcp list       # Codex
```

If `webpipe` is already listed, skip installation and go to Verify.

3. WebPipe currently ships as a local Python stdio MCP server only (there is no hosted remote server). The install step below covers that single path.
4. Look for an existing WebPipe API key in the environment variable `WEBPIPE_API_KEY`. Reuse it. Never print, log, or commit it.
5. If no key exists, tell the user to create one at https://webpipe.ai/api-keys.html and set it as `WEBPIPE_API_KEY`. Do not invent a key value.

### 2. Install the WebPipe MCP server

#### Local (Python stdio server)

Get the server code and install dependencies inside an isolated venv — do NOT
`pip install` into the shared/global interpreter (it is often used by other tools):

```bash
git clone https://github.com/tiejunshi/webpipe-mcp
cd webpipe-mcp
python -m venv .venv
.venv/Scripts/pip install fastmcp webpipe-sdk      # Windows
# .venv/bin/pip install fastmcp webpipe-sdk        # macOS / Linux
```

The MCP registration must point at the venv interpreter —
`<repo>/.venv/Scripts/python.exe` (Windows) or `<repo>/.venv/bin/python`
(macOS/Linux) — NOT the global `python`, or the imports will not resolve.

**Claude Code** (CLI):

```bash
claude mcp add webpipe --env WEBPIPE_API_KEY=your-webpipe-api-key \
  -- <venv-python> /absolute/path/to/webpipe-mcp/server.py
```

**Codex** (CLI):

```bash
codex mcp add webpipe --env WEBPIPE_API_KEY=your-webpipe-api-key \
  -- <venv-python> /absolute/path/to/webpipe-mcp/server.py
```

**JSON-configured agents.** The template below follows **Claude Desktop's config file format** (`mcpServers` + `command`/`args`/`env` — the de-facto stdio template most clients copy). Other agents use the same shape but different file locations — see the table after the template:

```json
{
  "mcpServers": {
    "webpipe": {
      "command": "<venv-python>",
      "args": ["/absolute/path/to/webpipe-mcp/server.py"],
      "env": { "WEBPIPE_API_KEY": "your-webpipe-api-key" }
    }
  }
}
```

| Agent | Config file | Notes |
| --- | --- | --- |
| Claude Code | `.mcp.json` (project) or `~/.claude.json` | Expands `${WEBPIPE_API_KEY}` automatically |
| WorkBuddy | `~/.workbuddy/mcp.json` (file name has no leading dot) | After saving, also trust/enable the server in the connectors UI — the config file alone is not enough |
| Cursor | `~/.cursor/mcp.json` (global) or `.cursor/mcp.json` (project) | Same `mcpServers` shape |
| Windsurf | `~/.codeium/windsurf/mcp_config.json` | Same `mcpServers` shape |
| Claude Desktop | Windows: `%APPDATA%\Claude\claude_desktop_config.json`; macOS: `~/Library/Application Support/Claude/claude_desktop_config.json` | Use the same JSON shape |
| Cline | Configure in Cline's "MCP Servers" panel | Panel accepts the same `mcpServers` JSON |
| VS Code (Copilot) | `.vscode/mcp.json` | Different schema: use `"servers"` as the top-level key instead of `"mcpServers"` |
| Gemini CLI | `~/.gemini/settings.json` | `mcpServers` key inside settings |

Clients that support environment-variable expansion (for example Claude Code) substitute `${WEBPIPE_API_KEY}` automatically. For clients that do not, put the literal key in place of `${WEBPIPE_API_KEY}` — warn the user that the key then sits in a plain config file.

Replace `<venv-python>` with the venv interpreter path and `/absolute/path/to/webpipe-mcp/server.py` with the real absolute path on this machine, and put the real key from the dashboard in place of `your-webpipe-api-key`.

### 3. Authorize the server inside the client (if required)

Some clients (WorkBuddy, for example) require an explicit trust/enable step in
their MCP/connectors management UI before the server is loaded — writing the
config file alone is not enough. Symptom: the config looks correct and the
server process launches, but no `webpipe_*` tools appear in the session. Fix:
open the client's MCP management screen and trust/enable `webpipe` there.

### 4. Authenticate

WebPipe uses a single API key. Precedence, highest first:

1. The explicit `api_key` argument passed to a tool call.
2. The environment variable `WEBPIPE_API_KEY`.

The client passes the key to the server as the environment variable `WEBPIPE_API_KEY`. GUI clients do not always inherit user-level environment variables — if `webpipe_verify_auth` reports a missing key although the variable is set, put the key directly into the `env` block of the client's MCP config.

Get a key at https://webpipe.ai/api-keys.html. Never print, echo, log, or commit it.

### 5. Verify

Prove the integration works with two real calls. Ask the host agent to run:

```text
webpipe_verify_auth()
```

Expected: `{"authenticated": true, ...}`. If it returns `❌ API key is invalid or expired`, the key is wrong or missing — fix `WEBPIPE_API_KEY` and re-run this before anything else.

Then one real scrape:

```text
webpipe_scrape(url="https://example.com")
```

Expected result: a non-error payload whose Markdown contains "Example Domain".

One successful `webpipe_verify_auth` is enough to declare setup complete. Optionally exercise `webpipe_map`, `webpipe_crawl`, and `webpipe_browser` on a real site to confirm the full tool surface.

### 6. Troubleshooting

Every tool returns errors as a readable string (never an exception), prefixed with ❌ auth/param/server, ⚠️ rate limit, 🌐 network, or 🔴 backend — each message includes a recommended recovery step. Common cases:

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `❌ API key is invalid or expired` | Key missing, wrong, or expired | Fix `WEBPIPE_API_KEY`; run `webpipe_verify_auth` to confirm; create a new key at https://webpipe.ai/api-keys.html |
| `⚠️ Rate limit exceeded (429)` | Rate limit hit | Wait and retry with backoff; lower `max_concurrency`/`limit` for crawls |
| `❌ Bad request` | Bad tool arguments | Re-check the required `url` and optional parameters |
| `🌐 Network error` | Outbound network or proxy issue reaching `api.web2json.ai` | Retry once after a short wait; verify the machine can reach `api.web2json.ai` |
| `❌ Resource not found or expired` | Crawl job older than 24 hours | Re-run the crawl |
| `webpipe` not listed / MCP connection failed | Wrong URL or command | Remove and re-add the server; verify the Python path and that `pip install` succeeded |
| Tools not visible after install | Client needs a restart or rescan | Restart the client, then re-run `claude mcp list` / `codex mcp list` |

Only report **WebPipe ready** when all of these are true:

```text
WebPipe MCP server registered in the host agent
webpipe_verify_auth returned authenticated: true
A live webpipe_scrape returned real content
```

If the server is registered but the host has not rescanned yet, say **"installed; restart/rescan required"** instead of claiming the tools are active.

## Tool reference (essentials)

Full input/output details live in the tool schema and in [server.py](https://github.com/tiejunshi/webpipe-mcp) docstrings (each documents an output example). Only decision-relevant essentials are repeated here.

- **webpipe_verify_auth** — no required args. Call first during setup and after any auth error.
- **webpipe_scrape** — `url` required. `formats`: `markdown` (default), `html`, `rawHtml`, `links`, `metadata`, `json`; `json_schema` guides JSON extraction; `use_proxy`/`proxy`/`cookie` for anti-bot sites.
- **webpipe_map** — `url` required. `limit` (default 5000), `search` keyword filter, `sitemap` include/only/exclude, `same_domain` (default true).
- **webpipe_crawl** — `url` required. `limit` (default 100, max 10000), `max_discovery_depth`, `include_paths`/`exclude_paths` regexes, `max_concurrency` (1-10), `scrape_formats`. Blocks until finished; results kept 24 hours. Always pass a finite `timeout` (e.g. 120) and a small `limit` — without `timeout` the call blocks for the whole crawl, and a large `limit` returns every page in one payload (context-overflow risk).
- **webpipe_browser** — three usage patterns:
  - One-shot: omit `session_id` — create, run `actions`, scrape, auto-close.
  - Persistent: `auto_close=false` — the response carries a `session_id`; pass it on the next call to keep browser state (login, cookies) across calls.
  - Reusable login: `session_name` + `auto_close=false` — save the logged-in state under a name and restore it later.
  - `actions` types: `click`, `fill`, `select`, `press`, `scroll`, `wait`, `wait_for`, `hover`, `check`, `screenshot`, `evaluate`.

## Escalation pattern

Match the tool to the job:

1. **`webpipe_verify_auth`** when authentication is unconfirmed.
2. **`webpipe_scrape`** when the URL is already known.
3. **`webpipe_map`** when the site is known but the right pages are not.
4. **`webpipe_crawl`** when many pages from a site or section are needed.
5. **`webpipe_browser`** when the page needs JavaScript interaction (login, clicks, form fills, screenshots).
