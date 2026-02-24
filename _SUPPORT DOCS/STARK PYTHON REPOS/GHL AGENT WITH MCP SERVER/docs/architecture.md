# Architecture — ghl-adk-agent-w-mcp-server-v1

**App Type:** Production ADK Agent + TypeScript MCP Server (GoHighLevel CRM integration)
**Languages:** Python (ADK agent) + TypeScript/Node.js (MCP server)
**Deploy:** Single Cloud Run container with Nginx + Supervisor

---

## System Overview

```
User Browser
    |
    | HTTPS (Cloud Run URL)
    v
[Nginx :8080] ← HTTP Basic Auth (admin/stark)
    |
    | proxy_pass (+ SSE streaming headers)
    v
[ADK Web UI :8000]  ←→  [MCP Server :9000]
  Python/Gemini          Node.js/TypeScript
  MCPToolset             ~215 GHL tools
       |                      |
       | StreamableHTTP        | axios
       | POST /mcp             v
       +---------------→ GHL CRM API
                    (services.leadconnectorhq.com)
```

**Supervisor** (PID 1 in Docker) manages three processes:
1. **nginx** on port 8080 — public-facing, auth, SSE proxy
2. **adk web** on port 8000 — hidden, ADK UI + agent runtime
3. **npm run start:http** on port 9000 — hidden, MCP server

---

## File Structure

```
ghl-adk-agent-w-mcp-server-v1/
├── ghl_mcp_agent/            ← Python ADK agent
│   ├── agent.py              ← Main agent (Gemini + MCPToolset)
│   ├── agent-Litellm.py      ← LiteLLM variant (OpenRouter)
│   ├── agent-org.py          ← .org backup of original
│   ├── __init__.py           ← exports root_agent
│   └── .adk/session.db       ← local ADK session state
├── jarvis_agent/             ← Secondary debug/research agent
│   ├── agent.py              ← google_search agent
│   ├── agent_gcs.py          ← GCS callable instructions variant
│   └── __init__.py
├── utils/                    ← Shared Python utilities
│   ├── gcs_utils.py          ← fetch_instructions(agent_name) from GCS
│   ├── context_utils.py      ← fetch_document(file_name) from GCS
│   └── focus_group_utils.py
├── ghl-mcp-server-moose-v1/  ← TypeScript MCP server
│   ├── src/
│   │   ├── http-server.ts    ← Main server (McpServer + Streamable HTTP)
│   │   ├── stdio-server.ts   ← Stdio variant (Claude Desktop)
│   │   ├── clients/
│   │   │   └── ghl-api-client.ts  ← Axios client for GHL API
│   │   ├── tools/            ← 19 tool category classes
│   │   │   ├── contact-tools.ts
│   │   │   ├── conversation-tools.ts
│   │   │   ├── blog-tools.ts
│   │   │   └── ... (17 more categories)
│   │   └── types/
│   │       └── ghl-types.ts  ← All TypeScript interfaces
│   ├── tests/                ← Jest tests
│   └── package.json
├── Dockerfile                ← Multi-runtime build (Python + Node.js)
├── nginx.conf                ← Reverse proxy + SSE streaming config
├── supervisord.conf          ← 3-process orchestration
├── deploy.sh                 ← gcloud run deploy script
└── requirements.txt          ← Python deps (google-adk, fastapi, etc.)
```

---

## Key Service Ports

| Service | Port | Visibility | Purpose |
|---------|------|------------|---------|
| Nginx | 8080 | Public (Cloud Run) | Auth + SSE proxy |
| ADK Web | 8000 | Internal only | Agent UI + runtime |
| MCP Server | 9000 | Internal only | GHL tool execution |

---

## Docker Build Stages

```dockerfile
FROM python:3.12-slim

# 1. System deps: Node.js 20 + Supervisor + Nginx + apache2-utils (htpasswd)
RUN apt-get install -y nodejs supervisor nginx apache2-utils

# 2. Python deps
RUN pip install -r requirements.txt

# 3. Node deps (package.json first for layer caching)
COPY ghl-mcp-server-moose-v1/package*.json ...
RUN npm install

# 4. Copy all app code
COPY . .

# 5. Build TypeScript → dist/
RUN npm run build

# 6. Configure Nginx (basic auth + SSE headers)
COPY nginx.conf /etc/nginx/nginx.conf
RUN htpasswd -bc /etc/nginx/.htpasswd admin stark

# 7. Configure Supervisor (3 processes)
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

ENV PORT=8080
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

---

## MCP Tool Categories (19 domains, ~215 tools)

| Category | Tool Count | Domain |
|----------|-----------|--------|
| contacts | 32 | CRM contacts, tasks, notes, tags, bulk ops |
| conversations | 21 | SMS, email, messages, recordings |
| blog | 7 | Posts, authors, categories, sites |
| opportunities | 10 | Pipelines, stages, status |
| calendar | 14 | Events, appointments, free slots |
| location | 24 | Sub-accounts, tags, custom fields |
| email | 5 | Campaigns, templates |
| emailISV | 1 | Email verification |
| socialMedia | 17 | Posts, accounts, OAuth |
| media | 3 | File upload, library |
| objects | 9 | Custom object schemas + records |
| associations | 10 | Relationship mapping |
| customFieldsV2 | 8 | Custom field CRUD |
| workflows | 1 | Discovery |
| surveys | 2 | Surveys + submissions |
| store | 18 | Shipping zones, rates, carriers |
| products | 10 | Products, prices, inventory |
| payments | 20 | Orders, transactions, subscriptions, coupons |
| invoices | 39 | Templates, recurring, estimates |
| utility | 2 | datetime calculator, math calculator |

---

## Data Flow (User Query → GHL API Response)

```
1. User types in ADK Web UI (browser → Nginx:8080)
2. Nginx auth check → proxy to ADK:8000
3. ADK sends query to Gemini model
4. Gemini decides which tool to call
5. ADK calls MCPToolset → POST http://localhost:9000/mcp
6. MCP server executes tool → axios → GHL API
7. GHL API returns data → MCP returns tool result
8. Gemini formats response → ADK → Nginx → User
```

SSE streaming is required at step 8 (agent thinking tokens stream back).
Nginx must have `proxy_buffering off` for this to work.
