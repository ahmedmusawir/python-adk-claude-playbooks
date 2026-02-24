# FastAPI ADK Wrapper — Deep Dive Reference

This is the definitive reference for building a FastAPI wrapper around a Google ADK Agent Bundle. This pattern is the bridge between any frontend (or N8N workflow) and an ADK backend.

---

## The Problem This Solves

ADK's native API is designed for agent-to-agent and internal use. It's verbose:

```
# To send one message, you need to:
1. Create a session: POST /apps/{app}/users/{user}/sessions/{id}
2. Run the agent:    POST /run  (with specific JSON shape)
3. Parse the result: iterate events[], find last model text part
4. Handle session expiry: 404 means session gone, start over
```

Frontend apps shouldn't need to know this. The wrapper abstracts it into:
```
POST /run_agent { agent_name, message, user_id, session_id? }
→ { response, session_id, agent_name, status }
```

---

## ADK API Contract (Internal — What Wrapper Calls)

### Session Creation
```
POST {adk_url}/apps/{app_name}/users/{user_id}/sessions/{session_id}
Body: {}
Response: 200 OK (session created)
```

### Run Agent
```
POST {adk_url}/run
Body: {
  "app_name": "jarvis_agent",
  "user_id": "user-tony-stark",
  "session_id": "session-1729354800",
  "new_message": {
    "role": "user",
    "parts": [{"text": "Your message here"}]
  }
}
Response: 200 OK → events[] array
          404    → session not found
```

### Fetch Session (for history)
```
GET {adk_url}/apps/{app_name}/users/{user_id}/sessions/{session_id}
Response: 200 OK → { events: [...], ... }
          404    → session not found
```

---

## ADK Events Array Structure

The `/run` endpoint returns an array of event objects. Understanding this is essential for parsing.

```json
[
  {
    "author": "USER",
    "content": {
      "role": "user",
      "parts": [{"text": "Hello!"}]
    }
  },
  {
    "author": "jarvis_agent",
    "content": {
      "role": "model",
      "parts": [
        {"functionCall": {...}},
        {"text": "The final answer is..."}
      ]
    }
  },
  {
    "author": "jarvis_agent",
    "content": {
      "role": "model",
      "parts": [{"text": "Here is my response."}]
    }
  }
]
```

**Parsing rules:**
1. Iterate `reversed(events)` — the last model event is the final answer
2. Filter for `content.role == "model"` (not `"user"` or function calls)
3. Iterate `reversed(parts)` — last text part is the response
4. Check `isinstance(part, dict) and "text" in part` — not all parts have text (some are function calls)

**Why reversed?** Tool-use agents generate intermediate model events (thinking, function calls). The final text response is always last.

---

## Wrapper API Contract (External — What Frontend Calls)

### POST /run_agent

**Request:**
```json
{
  "agent_name": "jarvis_agent",
  "message": "What's the status?",
  "user_id": "user-tony-stark",
  "session_id": "session-1729354800"
}
```

- `session_id` is optional — omit to start a new conversation
- `user_id` is the caller's persistent identifier (no auth, just grouping)

**Response (200):**
```json
{
  "response": "The status is nominal.",
  "session_id": "session-1729354800",
  "agent_name": "jarvis_agent",
  "status": "success"
}
```

- `session_id` in response may differ from request if auto-recovery triggered
- Always update your stored `session_id` from the response

**Error responses:**
- `404` — agent_name not in registry
- `4xx` — ADK returned an error (propagated)
- `500` — unexpected error

---

### POST /get_history

**Request:**
```json
{
  "agent_name": "jarvis_agent",
  "user_id": "user-tony-stark",
  "session_id": "session-1729354800"
}
```

**Response (200):**
```json
{
  "history": [
    {"role": "user", "content": "What's the status?"},
    {"role": "assistant", "content": "The status is nominal."}
  ]
}
```

- Returns `{"history": []}` if session not found (graceful)
- `role` is always `"user"` or `"assistant"` (not ADK's `"USER"` / `"MODEL"`)
- Only includes turns with text content (filters out tool/function-call-only events)

---

### GET /health

```json
{
  "status": "healthy",
  "agents": ["greeting_agent", "jarvis_agent", "calc_agent", "product_agent", "ghl_mcp_agent"]
}
```

---

### GET /agents

```json
{
  "agents": ["greeting_agent", "jarvis_agent", "calc_agent", "product_agent", "ghl_mcp_agent"]
}
```

---

## Session Lifecycle

```
Frontend                    Wrapper                     ADK Bundle

1. First message:
   POST /run_agent           session_id = None
   session_id: None    ───► or create_session()  ──────► POST /apps/.../sessions/session-123
                             session_id = "session-123" ◄── 200 OK
                                                        POST /run (session-123)
                             parse events[]      ◄──────── events[]
                       ◄──── response + "session-123"

2. Subsequent messages:
   POST /run_agent           session_id = "session-123"
   session_id: session-123 ─► validate it exists  ──────► POST /run (session-123)
                             parse events[]       ◄──────── events[]
                       ◄──── response + "session-123"

3. Session expired:
   POST /run_agent           POST /run (session-123)
   session_id: session-123 ─► 404 ← ADK Bundle
                             create_session()    ──────────► POST /sessions/session-456
                             POST /run (session-456) ──────► events[]
                             parse events[]      ◄────────── events[]
                       ◄──── response + "session-456"  ← NEW session_id returned!
```

---

## Deployment: Two-Service Architecture

The wrapper and the ADK bundle are **separate Cloud Run services**. They are deployed independently.

```
Service 1 — ADK Bundle (google-adk-n8n-hybrid-v2 or similar)
    URL: https://adk-bundle-prod-v2-[hash].us-east1.run.app
    Deploy: adk api_server . (ADK's built-in server)
    State: Supabase PostgreSQL for sessions
    Secrets: DB_URI, GCS credentials via Secret Manager

Service 2 — ADK Wrapper (this repo)
    URL: https://adk-wrapper-prod-v2-[hash].us-east1.run.app
    Deploy: uvicorn main:app (FastAPI)
    State: None (stateless)
    Config: ADK_BUNDLE_URL via --set-env-vars (not a secret)
```

Deploy order: **bundle first, then wrapper** (wrapper needs bundle URL).

---

## Adding a New Agent

1. Add the agent to the ADK bundle repo and redeploy the bundle
2. Add the agent name to `config.json` agents list:
   ```json
   "agents": ["existing_agent", "new_agent"]
   ```
3. Redeploy the wrapper: `./deploy.sh`

No Python code changes needed. The registry is built dynamically from config.json.

---

## Comparison: Wrapper vs Direct ADK Access

| Concern | Via Wrapper | Direct ADK |
|---------|-------------|------------|
| Session creation | Automatic | Manual |
| Session expiry | Auto-recovered | Error |
| Response parsing | Handled | Manual |
| History format | Clean role/content | Raw ADK events |
| Adding agents | config.json only | URL changes |
| Frontend changes on redeploy | None | Possible |
| Bundle URL exposure to frontend | No | Yes |

---

## When to Use Which ADK Endpoint

| Use Case | Endpoint |
|----------|----------|
| Conversational UI (chat) | `POST /run_agent` via wrapper |
| Load previous conversation | `POST /get_history` via wrapper |
| Check service health | `GET /health` via wrapper |
| Streaming responses (chat typing effect) | Direct `/run_sse` on ADK — not supported by this wrapper |
| Agent-to-agent calls | Direct ADK API (wrapper not needed) |

---

## httpx Timeout Reference

| Operation | Timeout | Reason |
|-----------|---------|--------|
| `create_session` | 10s | Fast DB write, no LLM |
| `run_agent_session` | 60s | LLM inference + tool calls |
| `get_history` | 30s | DB read, no LLM |

Increase agent timeout if agents use slow tools (web search, GCS, external APIs).
