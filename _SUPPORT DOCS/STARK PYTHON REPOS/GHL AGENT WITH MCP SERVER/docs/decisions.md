# Decisions — ghl-adk-agent-w-mcp-server-v1

Key architectural decisions with context, alternatives, and rationale.

---

## Decision 1: Multi-Runtime Docker Container (Python + Node.js)

**Choice:** Single Docker image containing Python 3.12 + Node.js 20 + Nginx + Supervisor

**Why:** The ADK agent is Python (required for google-adk). The MCP server is TypeScript/Node.js (existing codebase with 215+ tools). Rather than deploy two separate Cloud Run services, everything is bundled in one container.

**Alternatives:**
- Two Cloud Run services (ADK + MCP) — more isolated but adds network latency, two deploy targets, cross-service auth complexity
- Rewrite MCP server in Python — months of work, all tools already working in TypeScript

**Trade-off:** Larger image, more complex Dockerfile. But simpler deployment (one URL, one service, one deploy command) and zero network latency between ADK and MCP (localhost).

**How it works:** Supervisor manages 3 processes — Nginx (8080), ADK (8000), MCP (9000). Nginx is the only public-facing port.

---

## Decision 2: Streamable HTTP Transport (not SSE)

**Choice:** `StreamableHTTPServerTransport` + `POST /mcp` endpoint

**Why:** The old SSE-based MCP protocol was deprecated. It required permanent connections, `/sse` + separate message endpoints, complex lifecycle management. Streamable HTTP is stateless (one request = one response), simpler to debug, and better for production.

**Alternative:** SSE transport (old `@modelcontextprotocol/sdk` API with `Server` class, `setRequestHandler()`, `/sse` endpoint)

**Key difference:**
- Old: `server.setRequestHandler(CallToolRequestSchema, async (request) => { ... })`
- New: `mcpServer.registerTool(name, schema, async (params) => { ... })`

The new pattern auto-routes by tool name — no manual switch/routing needed.

---

## Decision 3: Error-as-Response (Never Throw)

**Choice:** MCP tool handler catches errors and returns `{ isError: true }` response instead of throwing

**Why:** When an LLM calls 10 tools in parallel and one throws an exception, the MCP server fails to return a response for that tool. The LLM sees tool call IDs with no matching responses — "tool_call_id missing" error. The conversation session is permanently broken ("session poisoning"). All subsequent messages fail.

**Solution:** Every tool handler wraps execution in try/catch. On failure, it returns:
```typescript
{ content: [{ type: "text", text: `Error: ${msg}` }], isError: true }
```

The LLM receives an error message for that tool instead of silence, and the conversation continues.

**Alternative:** Throw exceptions — simple but causes session corruption.

---

## Decision 4: 30-Second Timeout Per Tool

**Choice:** `Promise.race([executionPromise, timeoutPromise])` with 30s timeout in each tool handler

**Why:** GHL API calls occasionally hang (network issues, rate limits, slow endpoints). Without timeout, one hung tool call stalls the agent indefinitely. ADK has its own timeout (45s at Express level), but individual tool protection is needed.

**Consideration:** Some operations (bulk updates, async operations) might legitimately need more than 30s. Increase if needed for specific tool categories.

---

## Decision 5: Zod Schemas for Tool Input Validation

**Choice:** `inputSchema: { field: z.string().describe('...') }` using Zod

**Why:** The new `McpServer` API requires Zod schemas. The old `Server` API used JSON schemas (`type: 'object', properties: {...}`). Zod schemas are:
- More concise (no `type: 'object'` wrapper needed)
- Type-safe (TypeScript inference)
- Auto-validated (MCP SDK validates input before calling handler)

**Migration note:** `contact-tools.ts` was converted first as the proof-of-concept. The remaining 16 categories were converted incrementally.

---

## Decision 6: Nginx Basic Auth for ADK Web UI

**Choice:** Nginx HTTP Basic Auth (`auth_basic` + `htpasswd`) in front of ADK

**Why:** ADK's `adk web` command starts a UI with no built-in authentication. Exposing it directly on Cloud Run with `--allow-unauthenticated` would make it publicly accessible. Nginx adds a simple username/password gate.

**Credentials:** hardcoded in Dockerfile (`htpasswd -bc /etc/nginx/.htpasswd admin stark`). In production, this should be rotated or replaced with a proper auth mechanism.

**Alternative:** Cloud Run IAM (`--no-allow-unauthenticated`) — stronger but requires signed requests for every user; harder for casual access.

---

## Decision 7: Sequential Tool Execution (Enforced via Instruction)

**Choice:** System prompt explicitly instructs the agent to execute ONE tool at a time, never in parallel

**Why:** MCP tool parallelism causes session poisoning (see Decision 3). Rather than fix all possible parallel call scenarios in code, the agent is instructed not to do it. This is a pragmatic workaround — the real fix would require the ADK/MCP layer to handle partial failures gracefully.

**Agent instruction text:**
```
Execute tools ONE AT A TIME, never in parallel.
Wait for each tool response before calling the next tool.
NEVER call the same tool multiple times simultaneously.
```

---

## Decision 8: `tool_filter` for Selective MCP Tool Exposure

**Choice:** MCPToolset supports a `tool_filter` parameter (commented out but available)

**Why:** With 215+ tools, the agent's context window can get overloaded with tool definitions. Filtering to relevant tool subsets reduces token usage and makes the agent more focused.

**Future pattern:** Create specialized agents (Contact Agent with only contact tools, Marketing Agent with social/email/blog tools, etc.) each connecting to the same MCP server but filtered to their domain.

```python
MCPToolset(
    connection_params=StreamableHTTPConnectionParams(url="http://localhost:9000/mcp"),
    tool_filter=['contacts_get-contact', 'contacts_search-contacts', 'contacts_create-contact']
)
```

---

## Decision 9: locationId Injected in MCP Server, Never Exposed to Agent

**Choice:** `GHLApiClient` reads `GHL_LOCATION_ID` from env and injects it into every API call

**Why:** The `locationId` is a GHL concept that scopes all operations to a specific account. It never changes per-request. The agent instruction explicitly says "The locationId is already configured — you do NOT need to ask users for it." This keeps the UX clean (users never see internal IDs) and prevents errors from missing or wrong locationId.

**Secrets injection:** `--set-secrets="GHL_API_KEY=ghl-api-key:latest,GHL_LOCATION_ID=ghl-location-id:latest"` in Cloud Run deploy command.

---

## Decision 10: Gemini Model in Production vs LiteLLM During Testing

**Choice:** Production uses `model="gemini-2.5-flash"` (or gemini-3-flash-preview) directly. Testing used LiteLLM with OpenRouter/OpenAI.

**Model progression seen in files:**
- `agent.py` — production: `"gemini-3-flash-preview"` (direct Vertex AI)
- `agent-Litellm.py` — dev/test: OpenRouter (Gemini flash-lite, Qwen, Claude, GPT-5-mini via LiteLLM)

**Lab notes from ai-context files:**
- GPT-4o-mini — hallucinated tool names, too aggressive with parallel calls
- GPT-5-mini — much better tool calling, affordable
- OpenRouter + Claude Sonnet 3.5 — "Provider returned error" (OpenRouter proxy layer incompatible)
- Gemini — recommended for production (native Vertex AI, no proxy layer)

**Lesson:** Multi-provider LLM testing is valuable for dev. Always converge on Vertex AI Gemini for production.
