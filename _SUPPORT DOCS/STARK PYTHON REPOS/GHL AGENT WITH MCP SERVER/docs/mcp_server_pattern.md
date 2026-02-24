# MCP Server Pattern — Deep Dive

This document covers the complete pattern for building a TypeScript MCP server that ADK agents can connect to via Streamable HTTP.

---

## What Is This Pattern?

An MCP (Model Context Protocol) server lets you expose any API as a set of tools that LLM agents can call. ADK's `MCPToolset` acts as the bridge — the agent sees MCP tools as if they were native ADK `FunctionTool` functions.

```
ADK Agent (Python)
    |
    | MCPToolset → POST /mcp (Streamable HTTP)
    v
MCP Server (TypeScript/Node.js)
    |
    | GHLApiClient → axios
    v
External API (GoHighLevel CRM)
```

---

## Core Package

```json
// package.json
{
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.12.1",
    "express": "^5.1.0",
    "zod": "^3.x"
  }
}
```

---

## Old vs New SDK Pattern

This repo migrated from the old SSE-based pattern. Understanding both helps when reading older MCP server code.

### Old Pattern (SSE — Deprecated)
```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { SSEServerTransport } from "@modelcontextprotocol/sdk/server/sse.js";
import { CallToolRequestSchema } from "@modelcontextprotocol/sdk/types.js";

const server = new Server({ name: "my-server", version: "1.0" }, { capabilities: { tools: {} } });

// Manual routing — must handle ALL tools in one handler
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;
  if (name === "get_contact") { ... }
  else if (name === "create_contact") { ... }
});

// SSE endpoint — permanent connection
app.get("/sse", async (req, res) => {
  const transport = new SSEServerTransport("/messages", res);
  await server.connect(transport);
});
app.post("/messages", async (req, res) => { ... });

// JSON schema for tools
inputSchema: {
  type: 'object',
  properties: { email: { type: 'string', description: '...' } },
  required: ['email']
}
```

### New Pattern (Streamable HTTP — Current)
```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";

const mcpServer = new McpServer({ name: "my-server", version: "1.0.0" });

// Per-tool registration — auto-routing
mcpServer.registerTool(
  "get_contact",
  { description: "Get contact by ID", inputSchema: { contactId: z.string() } },
  async (params) => { ... }
);

// Single stateless endpoint
app.post("/mcp", async (req, res) => {
  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined,  // stateless
    enableJsonResponse: true,
  });
  res.on("close", () => transport.close());
  await mcpServer.connect(transport);
  await transport.handleRequest(req, res, req.body);
});

// Zod schema for tools
inputSchema: {
  email: z.string().describe("Contact email address"),
  phone: z.string().optional().describe("Contact phone number"),
}
```

---

## Complete Minimal MCP Server

```typescript
import express from "express";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { z } from "zod";

const app = express();
app.use(express.json());

const mcpServer = new McpServer({ name: "my-api-server", version: "1.0.0" });

// Register tools
mcpServer.registerTool(
  "hello_world",
  { description: "Say hello", inputSchema: { name: z.string().describe("Name to greet") } },
  async ({ name }) => ({
    content: [{ type: "text" as const, text: `Hello, ${name}!` }],
    structuredContent: { greeting: `Hello, ${name}!` },
  })
);

// MCP endpoint
app.post("/mcp", async (req, res) => {
  const transport = new StreamableHTTPServerTransport({ sessionIdGenerator: undefined, enableJsonResponse: true });
  res.on("close", () => transport.close());
  await mcpServer.connect(transport);
  await transport.handleRequest(req, res, req.body);
});

app.get("/health", (req, res) => res.json({ status: "healthy" }));

app.listen(9000, "0.0.0.0", () => console.log("MCP Server running on :9000"));
```

---

## Tool Class Pattern (for large APIs)

For APIs with many endpoints, organize tools into domain classes:

```typescript
// tools/contact-tools.ts
export class ContactTools {
  constructor(private apiClient: MyApiClient) {}

  getToolDefinitions(): any[] {
    return [
      {
        name: "search_contacts",
        description: "Search contacts by name, email, or phone.",
        inputSchema: {
          query: z.string().optional().describe("Search term"),
          limit: z.number().optional().describe("Max results"),
        },
      },
      {
        name: "get_contact",
        description: "Get contact details by ID.",
        inputSchema: { contactId: z.string().describe("Contact ID") },
      },
    ];
  }

  async executeTool(name: string, params: any): Promise<any> {
    switch (name) {
      case "search_contacts": return await this.apiClient.search(params);
      case "get_contact": return await this.apiClient.get(params.contactId);
      default: throw new Error(`Unknown tool: ${name}`);
    }
  }
}
```

```typescript
// http-server.ts — registration
const contactTools = new ContactTools(apiClient);

for (const tool of contactTools.getToolDefinitions()) {
  mcpServer.registerTool(
    tool.name,
    { description: tool.description, inputSchema: tool.inputSchema },
    async (params) => {
      try {
        const result = await contactTools.executeTool(tool.name, params);
        return { content: [{ type: "text" as const, text: JSON.stringify(result) }], structuredContent: result };
      } catch (error) {
        const msg = error instanceof Error ? error.message : String(error);
        return { content: [{ type: "text" as const, text: `Error: ${msg}` }], isError: true };
      }
    }
  );
}
```

---

## Standard Tool Response Format

```typescript
// Success
return {
  content: [{ type: "text" as const, text: JSON.stringify(result) }],
  structuredContent: result,  // for structured data access
};

// Error (use isError, never throw)
return {
  content: [{ type: "text" as const, text: `Error: ${message}` }],
  structuredContent: { error: message, tool: toolName, params },
  isError: true,
};
```

---

## ADK Side — Connecting to the MCP Server

```python
# agent.py
from google.adk.agents import Agent
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, StreamableHTTPConnectionParams

root_agent = Agent(
    name="my_agent",
    model="gemini-2.5-flash",
    instruction="...",
    tools=[
        MCPToolset(
            connection_params=StreamableHTTPConnectionParams(
                url="http://localhost:9000/mcp",
            ),
            # Optionally filter to only expose specific tools
            # tool_filter=["search_contacts", "get_contact"]
        )
    ],
)
```

---

## Known Issues & Gotchas

### 1. Session Poisoning from Parallel Tool Calls
**Symptom:** "tool_call_id missing" error; all subsequent messages fail
**Cause:** LLM calls N tools in parallel; one throws; LLM waits for N responses but gets N-1
**Fix:** Never throw in tool handlers — always return `isError: true`
**Prevention:** Add agent instruction: "Execute tools ONE AT A TIME, never in parallel"

### 2. ADK Tool Call Timeout (ASGI timeout)
**Symptom:** Agent freezes mid-conversation; ADK logs show timeout
**Cause:** Tool execution takes too long (API call hangs)
**Fix:** Wrap tool execution in `Promise.race([executionPromise, timeoutPromise])` with 30s timeout

### 3. Nginx SSE Buffering
**Symptom:** ADK streaming responses appear in chunks or not at all through Nginx
**Fix:** Add to nginx.conf location block:
```nginx
proxy_buffering off;
proxy_cache off;
proxy_read_timeout 300s;
```

### 4. MCP Server Startup Before ADK
**Symptom:** ADK agent fails to connect to MCP on startup
**Why:** In Docker with Supervisor, all 3 processes start simultaneously. ADK might try to connect before MCP is ready.
**Fix:** Supervisor's `autorestart=true` — ADK will retry on failure. `--min-instances=1` on Cloud Run prevents cold start race.

### 5. TypeScript Tool Schema — Zod Required
**Symptom:** `McpServer.registerTool()` fails with schema error
**Why:** New `McpServer` API only accepts Zod schemas, not JSON Schema objects
**Fix:** Convert all `inputSchema` from JSON schema format to Zod format

---

## Transport Options

| Transport | Use Case |
|-----------|----------|
| `StreamableHTTPServerTransport` | Production (ADK, web services) |
| `StdioServerTransport` | Claude Desktop (local stdio pipe) |

The same tool registration code works with both transports. Only the endpoint setup differs.

---

## TypeScript Project Setup

```json
// tsconfig.json essentials
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "dist",
    "strict": true
  }
}
```

```json
// package.json scripts
{
  "scripts": {
    "build": "tsc",
    "start:http": "node dist/http-server.js",
    "start:stdio": "node dist/stdio-server.js",
    "dev": "nodemon --exec ts-node src/http-server.ts"
  }
}
```

In Docker: `RUN npm run build` then `CMD npm run start:http`
