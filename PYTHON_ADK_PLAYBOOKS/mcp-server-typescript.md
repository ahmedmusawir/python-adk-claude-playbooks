# MCP Server: TypeScript

## What This Is For

Build a TypeScript MCP server that exposes any external API as tools for an ADK agent. Complete setup using the current Streamable HTTP transport (the old SSE API is deprecated).

Sourced from the GHL Agent + MCP Server repo — 215 tools across 19 domains in production on Cloud Run.

---

## Quick Reference — Minimal MCP Server

```typescript
// src/server.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import express from "express";
import { z } from "zod";

const server = new McpServer({
  name: "my-mcp-server",
  version: "1.0.0",
});

// Register a tool
server.tool(
  "get_greeting",
  "Generate a greeting for a given name",
  {
    name: z.string().describe("Person's name to greet"),
  },
  async (args) => {
    try {
      const greeting = `Hello, ${args.name}! Welcome to the MCP server.`;
      return { content: [{ type: "text", text: greeting }] };
    } catch (error) {
      // NEVER throw — return error as response
      return {
        content: [{ type: "text", text: `Error: ${(error as Error).message}` }],
        isError: true,
      };
    }
  }
);

// HTTP server
const app = express();
app.use(express.json());

app.post("/mcp", async (req, res) => {
  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined,  // ← stateless mode — required
  });
  res.on("close", () => transport.close());
  await server.connect(transport);
  await transport.handleRequest(req, res, req.body);
});

app.get("/health", (req, res) => res.json({ status: "ok" }));

app.listen(9000, () => console.log("MCP server running on :9000"));
```

---

## Old vs New SDK (Migration)

The old SSE-based transport breaks with current ADK. Use Streamable HTTP.

```typescript
// ❌ OLD — SSE transport (deprecated, breaks with ADK MCPToolset)
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { SSEServerTransport } from "@modelcontextprotocol/sdk/server/sse.js";
// Old pattern used server.tool() differently, SSE endpoint at /sse

// ✅ NEW — Streamable HTTP (current)
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
// New pattern: POST /mcp endpoint, McpServer class, Zod schemas
```

**Package:** `@modelcontextprotocol/sdk` — same package, different import paths.

---

## Core Pattern — Tool Registration

```typescript
server.tool(
  "tool_name",                    // Tool name (used by ADK for routing)
  "What this tool does",          // Description (model reads this to decide when to use it)
  {                               // Input schema — MUST use Zod (not raw JSON Schema)
    param1: z.string().describe("What this parameter is"),
    param2: z.number().optional().describe("Optional number"),
    param3: z.enum(["a", "b", "c"]).describe("Must be one of these values"),
  },
  async (args) => {
    // args is typed based on your Zod schema
    try {
      const result = await doSomething(args.param1);
      return {
        content: [{ type: "text", text: JSON.stringify(result, null, 2) }],
      };
    } catch (error) {
      // ← error as response, never throw
      return {
        content: [{ type: "text", text: `Failed: ${error.message}` }],
        isError: true,
      };
    }
  }
);
```

**Zod is required.** The new SDK doesn't accept raw JSON Schema objects. Always use `z.string()`, `z.number()`, `z.object()`, etc.

---

## Error-as-Response — Never Throw

**This is the most critical pattern for MCP servers.** If a tool handler throws an exception, the MCP SDK terminates the session. Subsequent tool calls from the same agent fail. This is called session poisoning.

```typescript
// ❌ Wrong — throws, poisons session
async (args) => {
  const result = await apiCall(args.id);  // throws on 404
  return { content: [{ type: "text", text: result }] };
}

// ✅ Right — catches all errors, returns error as response
async (args) => {
  try {
    const result = await apiCall(args.id);
    return { content: [{ type: "text", text: JSON.stringify(result) }] };
  } catch (error) {
    return {
      content: [{ type: "text", text: `Error calling API: ${error.message}` }],
      isError: true,
    };
  }
}
```

The `isError: true` flag tells the model the tool failed, so it can report to the user or try a different approach. Without it, the model might interpret an error string as a successful response.

---

## 30-Second Timeout Wrapper

ADK has an internal timeout for MCP tool calls. Wrap slow API calls:

```typescript
async function withTimeout<T>(
  promise: Promise<T>,
  timeoutMs: number = 30000,
  timeoutMessage: string = "Operation timed out"
): Promise<T> {
  const timeout = new Promise<never>((_, reject) =>
    setTimeout(() => reject(new Error(timeoutMessage)), timeoutMs)
  );
  return Promise.race([promise, timeout]);
}

// Usage in tool handler:
async (args) => {
  try {
    const result = await withTimeout(
      fetchContactFromCRM(args.contact_id),
      25000,
      "CRM request timed out after 25s"
    );
    return { content: [{ type: "text", text: JSON.stringify(result) }] };
  } catch (error) {
    return {
      content: [{ type: "text", text: error.message }],
      isError: true,
    };
  }
}
```

---

## Tool Category Class Pattern

For 20+ tools, organize by domain. One class per domain, registration loop at startup.

```typescript
// src/tools/contacts.ts
import { z } from "zod";
import { GHLApiClient } from "../api/client.js";

export class ContactsTools {
  constructor(private api: GHLApiClient) {}

  getToolDefinitions() {
    return [
      {
        name: "search_contacts",
        description: "Search for contacts by name, email, or phone",
        schema: {
          query: z.string().describe("Search term (name, email, or phone)"),
          limit: z.number().optional().default(10).describe("Max results"),
        },
        handler: this.searchContacts.bind(this),
      },
      {
        name: "get_contact",
        description: "Get full contact details by ID",
        schema: {
          contact_id: z.string().describe("Contact ID"),
        },
        handler: this.getContact.bind(this),
      },
      {
        name: "update_contact",
        description: "Update a contact's fields",
        schema: {
          contact_id: z.string().describe("Contact ID"),
          fields: z.record(z.string()).describe("Fields to update as key-value pairs"),
        },
        handler: this.updateContact.bind(this),
      },
    ];
  }

  private async searchContacts(args: { query: string; limit?: number }) {
    try {
      const results = await withTimeout(
        this.api.searchContacts(args.query, args.limit || 10)
      );
      return { content: [{ type: "text" as const, text: JSON.stringify(results) }] };
    } catch (error) {
      return { content: [{ type: "text" as const, text: `Search failed: ${error.message}` }], isError: true };
    }
  }

  private async getContact(args: { contact_id: string }) {
    try {
      const contact = await withTimeout(this.api.getContact(args.contact_id));
      return { content: [{ type: "text" as const, text: JSON.stringify(contact) }] };
    } catch (error) {
      return { content: [{ type: "text" as const, text: `Get contact failed: ${error.message}` }], isError: true };
    }
  }

  private async updateContact(args: { contact_id: string; fields: Record<string, string> }) {
    try {
      const result = await withTimeout(this.api.updateContact(args.contact_id, args.fields));
      return { content: [{ type: "text" as const, text: JSON.stringify(result) }] };
    } catch (error) {
      return { content: [{ type: "text" as const, text: `Update failed: ${error.message}` }], isError: true };
    }
  }
}
```

Registration loop in `server.ts`:
```typescript
// src/server.ts
import { ContactsTools } from "./tools/contacts.js";
import { ConversationsTools } from "./tools/conversations.js";
import { WorkflowsTools } from "./tools/workflows.js";

const apiClient = new GHLApiClient({ baseURL: "https://api.gohighlevel.com" });

const toolModules = [
  new ContactsTools(apiClient),
  new ConversationsTools(apiClient),
  new WorkflowsTools(apiClient),
];

// Register all tools
for (const module of toolModules) {
  for (const tool of module.getToolDefinitions()) {
    server.tool(tool.name, tool.description, tool.schema, tool.handler);
  }
}
```

---

## locationId Injection at API Client Layer

For multi-tenant APIs (like GHL), the tenant ID (locationId) should be injected by the server, not passed by the agent. The agent doesn't need to know about tenancy.

```typescript
// src/api/client.ts
import axios, { AxiosInstance } from "axios";

export class GHLApiClient {
  private locationId: string;
  private http: AxiosInstance;

  constructor(config: { baseURL: string }) {
    // Inject from environment — agent never sees these
    this.locationId = process.env.GHL_LOCATION_ID!;
    this.http = axios.create({
      baseURL: config.baseURL,
      headers: { Authorization: `Bearer ${process.env.GHL_API_KEY!}` },
    });
  }

  async searchContacts(query: string, limit: number = 10) {
    // locationId injected here, not passed by tool caller
    const resp = await this.http.get("/contacts/search", {
      params: { locationId: this.locationId, query, limit },
    });
    return resp.data;
  }

  async getContact(contactId: string) {
    const resp = await this.http.get(`/contacts/${contactId}`, {
      params: { locationId: this.locationId },
    });
    return resp.data;
  }

  async updateContact(contactId: string, fields: Record<string, string>) {
    const resp = await this.http.put(`/contacts/${contactId}`, fields, {
      params: { locationId: this.locationId },
    });
    return resp.data;
  }
}
```

This pattern prevents the agent from needing to track tenant IDs, and prevents prompt injection attacks where a user tries to manipulate which tenant's data is accessed.

---

## Complete server.ts

```typescript
// src/server.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import express, { Request, Response } from "express";
import { ContactsTools } from "./tools/contacts.js";
import { GHLApiClient } from "./api/client.js";

const server = new McpServer({
  name: "ghl-mcp-server",
  version: "1.0.0",
});

// Register tools
const apiClient = new GHLApiClient({ baseURL: "https://services.leadconnectorhq.com" });
const contactsTools = new ContactsTools(apiClient);
for (const tool of contactsTools.getToolDefinitions()) {
  server.tool(tool.name, tool.description, tool.schema, tool.handler);
}

// Express app
const app = express();
app.use(express.json());

// MCP endpoint
app.post("/mcp", async (req: Request, res: Response) => {
  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined,  // ← stateless — prevents session tracking bugs
  });

  res.on("close", () => transport.close());

  try {
    await server.connect(transport);
    await transport.handleRequest(req, res, req.body);
  } catch (error) {
    if (!res.headersSent) {
      res.status(500).json({ error: "Internal server error" });
    }
  }
});

// Health check
app.get("/health", (req: Request, res: Response) => {
  res.json({ status: "ok", server: "ghl-mcp-server" });
});

const PORT = parseInt(process.env.PORT || "9000");
app.listen(PORT, () => {
  console.log(`MCP server running on port ${PORT}`);
});
```

---

## ADK Side — Connecting to MCP Server

```python
from google.adk.agents import Agent
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, StreamableHTTPConnectionParams

mcp_tools = MCPToolset(
    connection_params=StreamableHTTPConnectionParams(
        url="http://localhost:9000/mcp",
    ),
    # Optional: expose only specific tools
    tool_filter=["search_contacts", "get_contact", "update_contact"],
)

root_agent = Agent(
    name="crm_agent",
    model="gemini-2.5-flash",
    instruction="""You are a CRM assistant. Use the available tools to help users
    manage their contacts and conversations.

    IMPORTANT: Execute tools ONE AT A TIME. Wait for each result before the next call.""",
    tools=[mcp_tools],
)
```

---

## TypeScript Project Setup

**package.json:**
```json
{
  "name": "my-mcp-server",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "build": "tsc",
    "start": "node dist/server.js",
    "dev": "ts-node src/server.ts"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0",
    "express": "^4.18.0",
    "zod": "^3.22.0",
    "axios": "^1.6.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.0",
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0"
  }
}
```

**tsconfig.json:**
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "dist",
    "strict": true,
    "esModuleInterop": true
  },
  "include": ["src/**/*"]
}
```

Build: `npm run build` → outputs to `dist/`

---

## Dockerfile (for Multi-Runtime Container)

The TypeScript MCP server compiles at build time in Docker:

```dockerfile
FROM node:20-slim AS mcp-builder
WORKDIR /mcp
COPY mcp-server/package*.json ./
RUN npm ci
COPY mcp-server/ .
RUN npm run build

# In the final multi-runtime image, copy the built output:
# COPY --from=mcp-builder /mcp/dist /app/mcp/dist
# COPY --from=mcp-builder /mcp/node_modules /app/mcp/node_modules
```

See `multi-runtime-docker.md` for the complete multi-stage Dockerfile.

---

## Gotchas

**1. sessionIdGenerator: undefined is required for stateless mode**
Without it, the transport tries to maintain session state, leading to tracking bugs where subsequent requests fail. Always set `sessionIdGenerator: undefined` for stateless MCP servers.

**2. Zod schemas only — no raw JSON Schema**
The new SDK's `server.tool()` accepts Zod schemas in the third argument, not plain JSON Schema objects. `z.string()`, not `{ type: "string" }`.

**3. Never throw in tool handlers**
A thrown exception terminates the session. Subsequent tool calls in the same agent session fail. Catch everything, return `{ isError: true }`.

**4. TypeScript must be compiled, not run directly**
ADK connects to the running HTTP server, so the TypeScript code must be compiled first (`npm run build`). In Docker, run `npm run build` during the image build step.

**5. ADK timeout is ~30 seconds**
ADK's MCPToolset disconnects if a tool call takes longer than ~30 seconds. Use the `withTimeout()` wrapper on slow API calls to fail fast with a useful error message.

**6. enableJsonResponse: true flag**
Some tool responses need structured JSON. Add `enableJsonResponse: true` to your server config if the model needs to parse JSON from tool responses rather than treat them as plain text.

---

## What's Next

- Connect MCP server to ADK agent → `adk-advanced-patterns.md`
- Run MCP server + ADK in one container → `multi-runtime-docker.md`
- Deploy to Cloud Run → `cloud-run-deployment.md`
