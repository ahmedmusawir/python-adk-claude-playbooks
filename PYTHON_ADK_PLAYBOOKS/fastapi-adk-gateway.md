# FastAPI ADK Gateway

## What This Is For

How to wrap ADK's complex multi-step session API into a single clean `/run_agent` endpoint that any client (Next.js, Streamlit, n8n) can call with one HTTP request.

This is the user's preferred production architecture. All agents exposed as REST APIs, consumed from Next.js frontends.

---

## The Problem ADK's Native API Creates

ADK's native API requires 3+ steps before an agent can run:

```
1. POST /apps/{app_id}/users/{user_id}/sessions     → create session
2. POST /apps/{app_id}/users/{user_id}/sessions/{id}/run  → run agent (returns SSE stream)
3. Parse SSE events to extract response text
4. Handle 404 errors if session expired
```

No client should have to do this. The FastAPI wrapper hides all of it behind one endpoint.

---

## Quick Reference — The /run_agent Endpoint

```python
# wrapper/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import httpx
import json
import uuid

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

ADK_BASE_URL = "http://localhost:8000"  # ADK api_server (internal)

class RunAgentRequest(BaseModel):
    agent_name: str
    message: str
    user_id: str = "default_user"
    session_id: str | None = None

class RunAgentResponse(BaseModel):
    response: str
    session_id: str

@app.post("/run_agent", response_model=RunAgentResponse)
async def run_agent(req: RunAgentRequest):
    session_id = req.session_id or f"{req.agent_name}-{req.user_id}-{uuid.uuid4().hex[:8]}"

    async with httpx.AsyncClient(timeout=120.0) as client:
        # Ensure session exists
        session_id = await ensure_session(client, req.agent_name, req.user_id, session_id)

        # Run agent
        response_text = await run_agent_turn(client, req.agent_name, req.user_id, session_id, req.message)

    return RunAgentResponse(response=response_text, session_id=session_id)
```

---

## The Full Implementation

```python
# wrapper/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import httpx
import json
import uuid
import os
import logging

logger = logging.getLogger(__name__)

app = FastAPI(title="ADK Wrapper")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

ADK_BASE_URL = os.getenv("ADK_BASE_URL", "http://localhost:8000")


class RunAgentRequest(BaseModel):
    agent_name: str
    message: str
    user_id: str = "default_user"
    session_id: str | None = None


class RunAgentResponse(BaseModel):
    response: str
    session_id: str


@app.get("/health")
async def health():
    return {"status": "ok"}


@app.post("/run_agent", response_model=RunAgentResponse)
async def run_agent_endpoint(req: RunAgentRequest):
    session_id = req.session_id or f"{req.agent_name}-{req.user_id}-{uuid.uuid4().hex[:8]}"

    async with httpx.AsyncClient(timeout=120.0) as client:
        session_id = await ensure_session(client, req.agent_name, req.user_id, session_id)
        response_text, session_id = await run_agent_turn(client, req.agent_name, req.user_id, session_id, req.message)

    return RunAgentResponse(response=response_text, session_id=session_id)


async def ensure_session(
    client: httpx.AsyncClient,
    agent_name: str,
    user_id: str,
    session_id: str
) -> str:
    """Create ADK session if it doesn't exist. Returns session_id."""
    url = f"{ADK_BASE_URL}/apps/{agent_name}/users/{user_id}/sessions/{session_id}"
    resp = await client.post(url, json={})

    if resp.status_code in (200, 201):
        return session_id

    # Session already exists — that's fine
    if resp.status_code == 409:
        return session_id

    # Any other error
    logger.warning(f"Session creation returned {resp.status_code}: {resp.text}")
    return session_id


async def run_agent_turn(
    client: httpx.AsyncClient,
    agent_name: str,
    user_id: str,
    session_id: str,
    message: str,
) -> tuple[str, str]:
    """Send message to ADK agent. Returns (response_text, session_id).
    session_id may change if 404 recovery creates a new session."""
    url = f"{ADK_BASE_URL}/apps/{agent_name}/users/{user_id}/sessions/{session_id}/run"

    payload = {
        "new_message": {
            "role": "user",
            "parts": [{"text": message}]
        }
    }

    try:
        resp = await client.post(url, json=payload)

        # Session 404 — auto-recovery
        if resp.status_code == 404:
            logger.info(f"Session 404, creating new session and retrying")
            session_id = f"{agent_name}-{user_id}-{uuid.uuid4().hex[:8]}"
            await ensure_session(client, agent_name, user_id, session_id)
            resp = await client.post(
                f"{ADK_BASE_URL}/apps/{agent_name}/users/{user_id}/sessions/{session_id}/run",
                json=payload
            )

        resp.raise_for_status()
        events = resp.json()
        return extract_text_from_events(events), session_id

    except httpx.TimeoutException:
        raise HTTPException(status_code=504, detail="Agent request timed out")
    except Exception as e:
        logger.error(f"Agent run error: {e}")
        raise HTTPException(status_code=500, detail=str(e))


def extract_text_from_events(events: list) -> str:
    """Parse ADK events array, return concatenated text response."""
    text_parts = []

    for event in events:
        # ADK events structure: {"content": {"parts": [{"text": "..."}]}}
        content = event.get("content", {})
        if not content:
            continue

        role = content.get("role", "")
        if role == "model":
            parts = content.get("parts", [])
            for part in parts:
                if "text" in part and part["text"]:
                    text_parts.append(part["text"])

    return "".join(text_parts).strip()
```

---

## Session 404 Auto-Recovery

ADK sessions can expire or be lost (instance restart, memory pressure, new deployment). The `run_agent_turn` function above handles this automatically: it detects a 404, creates a fresh session, retries the request, and returns the new `session_id` to the caller. The user sees no error.

Key detail: `run_agent_turn` returns `(response_text, session_id)` — the `session_id` may differ from what was passed in if recovery triggered. The endpoint always returns the current session_id in the response, and callers must store it for the next turn.

---

## History Endpoint — Normalize ADK Events to Chat Format

Frontend apps want `[{role, content}]` format, not raw ADK events. The wrapper normalizes it.

```python
@app.get("/history/{agent_name}/{user_id}/{session_id}")
async def get_history(agent_name: str, user_id: str, session_id: str):
    """Return conversation history in {role, content} format."""
    url = f"{ADK_BASE_URL}/apps/{agent_name}/users/{user_id}/sessions/{session_id}"

    async with httpx.AsyncClient(timeout=30.0) as client:
        resp = await client.get(url)

        if resp.status_code == 404:
            return {"messages": []}

        resp.raise_for_status()
        session_data = resp.json()

    messages = normalize_history(session_data.get("events", []))
    return {"messages": messages, "session_id": session_id}


def normalize_history(events: list) -> list[dict]:
    """Convert ADK events to [{role: 'user'|'assistant', content: str}]."""
    messages = []

    for event in events:
        content = event.get("content", {})
        if not content:
            continue

        role = content.get("role", "")
        parts = content.get("parts", [])

        text = " ".join(p.get("text", "") for p in parts if "text" in p).strip()

        if text and role in ("user", "model"):
            messages.append({
                "role": "assistant" if role == "model" else "user",
                "content": text,
            })

    return messages
```

---

## Agent Registry — config.json + config.py

When the wrapper serves multiple agents, use a config file to register them. Different environments (dev vs prod) can point to different ADK base URLs.

```json
{
  "development": {
    "adk_base_url": "http://localhost:8000",
    "agents": {
      "search_agent": "Search the web for information",
      "email_agent": "Generate and edit email templates",
      "crm_agent": "Interact with CRM data"
    }
  },
  "production": {
    "adk_base_url": "https://adk-bundle.run.app",
    "agents": {
      "search_agent": "Search the web for information",
      "email_agent": "Generate and edit email templates",
      "crm_agent": "Interact with CRM data"
    }
  }
}
```

```python
# config.py
import json
import os
from pathlib import Path

def load_config():
    env = os.getenv("APP_ENV", "development")
    config_path = Path(__file__).parent / "config.json"
    with open(config_path) as f:
        all_config = json.load(f)
    return all_config.get(env, all_config["development"])

CONFIG = load_config()
ADK_BASE_URL = CONFIG["adk_base_url"]
AGENTS = CONFIG["agents"]
```

Add agent validation at the top of `/run_agent`:
```python
@app.post("/run_agent", response_model=RunAgentResponse)
async def run_agent_endpoint(req: RunAgentRequest):
    if req.agent_name not in AGENTS:
        raise HTTPException(status_code=400, detail=f"Unknown agent: {req.agent_name}")

    session_id = req.session_id or f"{req.agent_name}-{req.user_id}-{uuid.uuid4().hex[:8]}"

    async with httpx.AsyncClient(timeout=120.0) as client:
        session_id = await ensure_session(client, req.agent_name, req.user_id, session_id)
        response_text, session_id = await run_agent_turn(client, req.agent_name, req.user_id, session_id, req.message)

    return RunAgentResponse(response=response_text, session_id=session_id)
```

---

## Two-Service Architecture

In production, the wrapper and ADK bundle are separate Cloud Run services. The wrapper is public; the ADK bundle is internal.

```
Next.js / Streamlit
    ↓ POST /run_agent
ADK Wrapper (Cloud Run, public, port 8080)
    ↓ POST /apps/{agent}/users/{user}/sessions/{id}/run
ADK Bundle (Cloud Run, internal, port 8000)
    ↓
Agent → Vertex AI / MCP Server / Tools
```

In the **same container** (multi-runtime Docker), the ADK bundle runs internally on port 8000, wrapper on port 8080, and Nginx is the public face. See `multi-runtime-docker.md`.

For separate Cloud Run services, grant the wrapper's service account `roles/run.invoker` on the bundle service:

```bash
gcloud run services add-iam-policy-binding adk-bundle \
  --region us-east1 \
  --member="serviceAccount:wrapper-sa@project.iam.gserviceaccount.com" \
  --role="roles/run.invoker"
```

Then set `ADK_BASE_URL` in the wrapper's env vars to the bundle's Cloud Run URL.

---

## Utility Endpoints

```python
@app.get("/agents")
async def list_agents():
    """List available agents and their descriptions."""
    return {"agents": AGENTS}

@app.delete("/session/{agent_name}/{user_id}/{session_id}")
async def delete_session(agent_name: str, user_id: str, session_id: str):
    """Delete a session (for 'Start Over' buttons)."""
    url = f"{ADK_BASE_URL}/apps/{agent_name}/users/{user_id}/sessions/{session_id}"
    async with httpx.AsyncClient(timeout=10.0) as client:
        resp = await client.delete(url)
    if resp.status_code in (200, 204, 404):
        return {"deleted": True}
    raise HTTPException(status_code=500, detail="Could not delete session")
```

---

## How Next.js Consumes This API

```typescript
// lib/agent.ts

const WRAPPER_URL = process.env.NEXT_PUBLIC_ADK_WRAPPER_URL;

interface AgentResponse {
  response: string;
  session_id: string;
}

export async function runAgent(
  agentName: string,
  message: string,
  sessionId: string | null,
  userId: string
): Promise<AgentResponse> {
  const resp = await fetch(`${WRAPPER_URL}/run_agent`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      agent_name: agentName,
      message,
      user_id: userId,
      session_id: sessionId,
    }),
  });

  if (!resp.ok) throw new Error(`Agent error: ${resp.status}`);
  return resp.json();
}

// React component usage:
// const { response, session_id } = await runAgent("search_agent", userInput, sessionId, userId);
// setSessionId(session_id);  // Store for next turn
// setMessages(prev => [...prev, { role: "assistant", content: response }]);
```

---

## Deployment

**Dockerfile:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD uvicorn main:app --host 0.0.0.0 --port $PORT
```

**requirements.txt:**
```
fastapi
uvicorn[standard]
httpx
pydantic
```

**deploy.sh:**
```bash
gcloud run deploy adk-wrapper \
  --source . \
  --region us-east1 \
  --allow-unauthenticated \
  --memory 512Mi \
  --cpu 1 \
  --min-instances 1 \
  --set-env-vars="ADK_BASE_URL=https://adk-bundle.run.app,APP_ENV=production" \
  --service-account "wrapper-sa@project.iam.gserviceaccount.com"
```

---

## Adding API Key Auth (Production)

Add a simple middleware to require an API key from all callers:

```python
from fastapi import Request
from fastapi.responses import JSONResponse
import os

API_KEY = os.getenv("WRAPPER_API_KEY")

@app.middleware("http")
async def api_key_middleware(request: Request, call_next):
    # Skip auth for health check
    if request.url.path == "/health":
        return await call_next(request)

    if API_KEY:
        provided_key = request.headers.get("X-API-Key")
        if provided_key != API_KEY:
            return JSONResponse(status_code=401, content={"detail": "Invalid API key"})

    return await call_next(request)
```

Store `WRAPPER_API_KEY` in Secret Manager and inject via `--set-secrets`.

---

## Gotchas

**1. ADK events array, not SSE stream**
`adk api_server` returns a JSON array of events from the `/run` endpoint (not an SSE stream). Parse it as `resp.json()` — don't try to stream it.

**2. Session 404 happens regularly**
Cloud Run instance restarts, memory pressure, new deployments — sessions disappear. The 404 auto-recovery is not optional for production. Without it, users lose their conversations on every deployment.

**3. httpx timeout must be generous**
Default httpx timeout is 5 seconds. Agent calls with multiple tool invocations take 30-120 seconds. Set `timeout=120.0` minimum. 300 for complex agents.

**4. agent_name must match the folder name**
ADK uses the agent's `name` field as the app name in routing. If your agent is `root_agent = Agent(name="search_agent", ...)` in a folder called `search_agent/`, the URL path is `/apps/search_agent/...`. Mismatch = 404.

**5. session_id returned by wrapper may change**
If 404 recovery triggered, the wrapper creates a new session_id. The caller must use the `session_id` from the response, not the one it sent. Always store the returned session_id.

---

## What's Next

- Multi-agent pipelines inside the ADK bundle → `adk-advanced-patterns.md`
- Deploy the full two-service architecture → `cloud-run-deployment.md`
- Build Streamlit UI that calls this wrapper → `streamlit-patterns.md`
