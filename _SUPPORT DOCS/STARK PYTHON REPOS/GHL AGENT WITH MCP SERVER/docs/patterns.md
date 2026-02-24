# Patterns — ghl-adk-agent-w-mcp-server-v1

Copy-pasteable patterns extracted from this repo.

---

## Pattern 1: ADK Agent with MCPToolset (Streamable HTTP)

```python
# agent.py
from google.adk.agents import Agent
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, StreamableHTTPConnectionParams

root_agent = Agent(
    name="ghl_mcp_agent",
    model="gemini-2.5-flash",
    description="Agent connected to external MCP server",
    instruction=get_instructions,  # function, not function()
    tools=[
        MCPToolset(
            connection_params=StreamableHTTPConnectionParams(
                url="http://localhost:9000/mcp",
            ),
            # Optional: filter which tools are exposed to this agent
            # tool_filter=['contacts_get-contact', 'contacts_create-contact']
        )
    ],
)
```

**Key:** `StreamableHTTPConnectionParams` (not SSE). MCP server must expose a `POST /mcp` endpoint.

---

## Pattern 2: MCPToolset with LiteLLM (Multi-Provider)

```python
# agent-Litellm.py
from google.adk.agents import Agent
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, StreamableHTTPConnectionParams
from google.adk.models.lite_llm import LiteLlm

lite_llm_client = LiteLlm(
    model="openrouter/google/gemini-2.5-flash-lite",
    api_key=os.getenv("OPENROUTER_API_KEY"),
)

root_agent = Agent(
    name="ghl_mcp_agent",
    model=lite_llm_client,  # LiteLlm instance, not string
    tools=[MCPToolset(connection_params=StreamableHTTPConnectionParams(url="http://localhost:9000/mcp"))],
)
```

**Use when:** Testing non-Vertex models (OpenRouter, OpenAI) during development.

---

## Pattern 3: TypeScript MCP Server — Tool Category Class

```typescript
// src/tools/contact-tools.ts
import { z } from "zod";
import { GHLApiClient } from "../clients/ghl-api-client.js";

export class ContactTools {
  constructor(private ghlClient: GHLApiClient) {}

  getToolDefinitions(): any[] {
    return [
      {
        name: "search_contacts",
        description: "Search for contacts by name, email, phone, or other criteria.",
        inputSchema: {
          query: z.string().optional().describe("Search query string"),
          email: z.string().optional().describe("Filter by email"),
          limit: z.number().optional().describe("Max results (default: 25)"),
        },
      },
      // ... more tools
    ];
  }

  async executeTool(name: string, params: any): Promise<any> {
    switch (name) {
      case "search_contacts":
        return await this.ghlClient.searchContacts(params);
      // ...
    }
  }
}
```

**Pattern:** `getToolDefinitions()` returns tool list with Zod `inputSchema`.
`executeTool(name, params)` routes by name to private API call methods.

---

## Pattern 4: MCP Server Registration Loop (Streamable HTTP)

```typescript
// src/http-server.ts — the registration loop used for every tool category
const toolDefs = this.contactTools.getToolDefinitions();

for (const tool of toolDefs) {
  this.mcpServer.registerTool(
    tool.name,
    { description: tool.description, inputSchema: tool.inputSchema },
    async (params: any) => {
      const startTime = Date.now();

      try {
        // 30-second timeout wrapper
        const timeoutPromise = new Promise((_, reject) => {
          setTimeout(() => reject(new Error("Tool execution timeout after 30s")), 30000);
        });
        const executionPromise = this.contactTools.executeTool(tool.name, params);
        const result = await Promise.race([executionPromise, timeoutPromise]);

        return {
          content: [{ type: "text" as const, text: JSON.stringify(result) }],
          structuredContent: result,
        };
      } catch (error) {
        // Return error as response — NEVER throw (would poison session)
        const errorMessage = error instanceof Error ? error.message : String(error);
        return {
          content: [{ type: "text" as const, text: `Error: ${errorMessage}` }],
          structuredContent: { error: errorMessage, tool: tool.name, params },
          isError: true,
        };
      }
    }
  );
}
```

**Two critical patterns here:**
1. **30s timeout** — prevents GHL API hangs from stalling the agent
2. **Error-as-response** — `isError: true` instead of throwing; throwing breaks the entire session

---

## Pattern 5: MCP HTTP Server Entrypoint (Express + McpServer)

```typescript
// src/http-server.ts
import express from "express";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";

class GHLMCPHttpServer {
  private mcpServer = new McpServer({ name: "ghl-mcp-server", version: "1.0.0" });
  private app = express();

  // Endpoint that ADK MCPToolset connects to
  setupRoutes() {
    this.app.post("/mcp", async (req, res) => {
      const transport = new StreamableHTTPServerTransport({
        sessionIdGenerator: undefined,  // stateless mode
        enableJsonResponse: true,
      });

      res.on("close", () => transport.close());

      await this.mcpServer.connect(transport);
      await transport.handleRequest(req, res, req.body);
    });

    // Health check
    this.app.get("/health", (req, res) => res.json({ status: "healthy" }));
  }

  async start() {
    this.app.listen(this.port, "0.0.0.0");
  }
}
```

---

## Pattern 6: Supervisor Multi-Process Config

```ini
# supervisord.conf
[supervisord]
nodaemon=true          # stays in foreground (required for Docker CMD)
logfile=/dev/null      # logs to stdout instead

[program:nginx]
command=/usr/sbin/nginx -g "daemon off;"
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0

[program:adk]
directory=/app
command=adk web --port 8000 --host 0.0.0.0
autostart=true
autorestart=true

[program:mcp]
directory=/app/ghl-mcp-server-moose-v1
command=npm run start:http
environment=PORT=9000
autostart=true
autorestart=true
```

**Use when:** Running multiple processes in a single Docker container.

---

## Pattern 7: Nginx — SSE Streaming Proxy + Basic Auth

```nginx
# nginx.conf
events {}

http {
  server {
    listen 8080;

    # HTTP Basic Auth (protects ADK UI from public access)
    auth_basic "Restricted Area";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
      proxy_pass http://127.0.0.1:8000;
      proxy_http_version 1.1;

      # Required for ADK WebSocket/SSE upgrades
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;

      # CRITICAL for SSE/streaming — disable all buffering
      proxy_buffering off;
      proxy_cache off;
      proxy_read_timeout 300s;   # 5 min for slow agent responses
      proxy_send_timeout 300s;
    }
  }
}
```

**Create the password file in Dockerfile:**
```dockerfile
RUN htpasswd -bc /etc/nginx/.htpasswd admin stark
# -b = use password from command line, -c = create file
```

---

## Pattern 8: GCS Callable Instructions (Confirmed Cross-Repo)

```python
# utils/gcs_utils.py
from google.cloud import storage

BUCKET_NAME = "adk-agent-context-ninth-potion-455712-g9"
BASE_FOLDER = "ADK_Agent_Bundle_1"

def fetch_instructions(agent_name: str) -> str:
    storage_client = storage.Client()
    bucket = storage_client.bucket(BUCKET_NAME)
    file_path = f"{BASE_FOLDER}/{agent_name}/{agent_name}_instructions.txt"
    blob = bucket.blob(file_path)
    return blob.download_as_text(encoding='utf-8')

# agent.py usage — pass function, NOT function()
root_agent = Agent(
    instruction=get_live_instructions,  # ← callable, fetched fresh per run
)

def get_live_instructions(ctx) -> str:
    return fetch_instructions("my_agent")
```

---

## Pattern 9: Agent Safety Instructions for MCP Tools

These system prompt instructions prevent session poisoning from parallel tool call failures:

```
=== CRITICAL: TOOL EXECUTION RULES ===

1. Execute tools ONE AT A TIME, never in parallel
   - Wait for each tool response before calling the next tool
   - NEVER call the same tool multiple times simultaneously

2. For batch operations, process sequentially
   - Report progress after each: "Contact 1 of 3... ✅ Done. Now contact 2..."

3. Handle errors gracefully
   - If a tool fails, ask user what to do (retry, skip, abort)
   - NEVER retry automatically without asking first

4. Confirm destructive operations
   - Before deleting: ask for confirmation
   - Before bulk updates: confirm scope and count
```

**Why this matters:** When an LLM calls 10 tools in parallel and one fails, LLM expects responses for ALL 10. MCP server error on one breaks the entire response chain → session must be restarted.

---

## Pattern 10: Protocol Phoenix — UI Error Recovery

A named protocol in the system prompt for recovering from ADK display bugs:

```
=== PROTOCOL PHOENIX ===
Trigger: User types exactly: again?

Action:
1. DO NOT treat as a new query
2. Access session memory — locate result from most recent successful tool call
3. Re-present that exact result immediately
4. If no recent result: "My apologies, I don't have a recent result in memory."
```

**Use case:** ADK UI sometimes shows "unknown error" after successful tool call but doesn't display the response. User types "again?" to get it re-shown without re-executing.

---

## Pattern 11: HITL Flight Plan (Protocol 10)

Human-in-the-loop review before any create/update operation:

```
=== PROTOCOL 10: MANDATORY DATA REVIEW (CREATE/UPDATE OPS) ===
Trigger: ANY request to CREATE or UPDATE an entity.

Procedure:
1. STOP — Do NOT execute the tool immediately
2. Present a "Review Screen" containing:
   - Target Action: (e.g., "Creating New Product")
   - Required Data: list fields needed
   - Optional Data: helpful optional fields
   - Constraints: explicit rules (e.g., "Price must be in CENTS")
3. IGNORE system fields: never ask for locationId, token, or apiKey
4. EXECUTE only after user confirms
```

---

## Pattern 12: Cloud Run Deploy — Multi-Service Container

```bash
# deploy.sh
gcloud run deploy "$SERVICE_NAME" \
  --image "$IMAGE_NAME" \
  --region "us-east1" \
  --platform managed \
  --allow-unauthenticated \
  --service-account="stark-vertex-ai@${PROJECT_ID}.iam.gserviceaccount.com" \
  --set-secrets="GHL_API_KEY=ghl-api-key:latest,GHL_LOCATION_ID=ghl-location-id:latest" \
  --set-env-vars="GOOGLE_CLOUD_LOCATION=global,GOOGLE_GENAI_USE_VERTEXAI=TRUE" \
  --min-instances=1 \
  --timeout=600 \
  --cpu=2 \
  --memory=2Gi \
  --no-cpu-throttling \
  --cpu-boost
```

**Notes:**
- `--no-cpu-throttling` + `--cpu-boost` — required for streaming/SSE workloads
- `--min-instances=1` — prevent cold start (agent takes time to connect to MCP)
- `--timeout=600` — long timeout for extended agent conversations
- `--allow-unauthenticated` — Nginx handles auth, Cloud Run stays open

---

## Pattern 13: GHL API Client (TypeScript Axios)

```typescript
// src/clients/ghl-api-client.ts
import axios, { AxiosInstance } from 'axios';
import { GHLConfig } from '../types/ghl-types.js';

export class GHLApiClient {
  private client: AxiosInstance;
  private config: GHLConfig;

  constructor(config: GHLConfig) {
    this.config = config;
    this.client = axios.create({
      baseURL: config.baseUrl,  // https://services.leadconnectorhq.com
      headers: {
        'Authorization': `Bearer ${config.accessToken}`,
        'Version': config.version,  // '2021-07-28'
        'Content-Type': 'application/json',
      },
    });
  }

  // locationId injected into every request — never exposed to agent/user
  async searchContacts(params: any) {
    return this.client.get('/contacts/', {
      params: { locationId: this.config.locationId, ...params }
    });
  }
}
```

**Key:** `locationId` is injected by the client from env vars — agent never needs to know or ask for it.

---

## Pattern 14: MCP Utility Tools (domain-agnostic)

Two utility tools registered differently — using `server.tool()` directly (not via class):

```typescript
// src/tools/utility-tools.ts
export function registerUtilityTools(server: McpServer): void {
  server.tool(
    "calculate_future_datetime",
    "Calculate a future date/time for use with GHL API (ISO 8601 output).",
    {
      days: z.number().default(0).describe("Days from now"),
      hours: z.number().default(0).describe("Hour of day (24hr) or hours to add"),
      useAbsoluteTime: z.boolean().default(true),
    },
    async ({ days, hours, minutes, useAbsoluteTime }) => {
      // ... calculation
      return { content: [{ type: "text", text: JSON.stringify(result) }], structuredContent: result };
    }
  );
}
```

**Pattern:** Domain-agnostic tools use `server.tool()` directly. Domain tools use class pattern.
