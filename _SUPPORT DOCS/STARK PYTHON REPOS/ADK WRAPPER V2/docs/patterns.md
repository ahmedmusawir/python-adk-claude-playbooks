# Patterns: ADK Wrapper (google-adk-wrapper-v2)

Copy-pasteable patterns for building FastAPI wrappers around ADK bundles.

---

## Pattern 1: The Core `/run_agent` Endpoint

The primary pattern — accept a simplified request, manage session, call ADK, return clean response.

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
import httpx
import time

app = FastAPI(title="ADK Agent Gateway", version="1.0.0")

class AgentRequest(BaseModel):
    agent_name: str
    message: str
    user_id: str
    session_id: Optional[str] = None

class AgentResponse(BaseModel):
    response: str
    session_id: str
    agent_name: str
    status: str

@app.post("/run_agent", response_model=AgentResponse)
async def run_agent(request: AgentRequest):
    if request.agent_name not in AGENT_REGISTRY:
        raise HTTPException(status_code=404, detail=f"Agent '{request.agent_name}' not found")

    agent_url = AGENT_REGISTRY[request.agent_name]

    try:
        session_id = request.session_id or await create_session(agent_url, request.user_id, request.agent_name)
        response_text, effective_session_id = await run_agent_session(
            agent_url, request.message, request.user_id, request.agent_name, session_id
        )
        return AgentResponse(
            response=response_text,
            session_id=effective_session_id,
            agent_name=request.agent_name,
            status="success",
        )
    except httpx.HTTPStatusError as e:
        raise HTTPException(status_code=e.response.status_code, detail=f"ADK error: {e.response.text}")
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

---

## Pattern 2: Session Creation

```python
async def create_session(agent_url: str, user_id: str, app_name: str) -> str:
    session_id = f"session-{int(time.time())}"    # epoch-based, unique enough
    async with httpx.AsyncClient(timeout=10.0) as client:
        response = await client.post(
            f"{agent_url}/apps/{app_name}/users/{user_id}/sessions/{session_id}",
            headers={"Content-Type": "application/json"},
            json={}
        )
        response.raise_for_status()
    return session_id
```

**ADK session creation URL format:**
```
POST {adk_base_url}/apps/{app_name}/users/{user_id}/sessions/{session_id}
Body: {}
```

---

## Pattern 3: ADK `/run` Call + Event Parsing

ADK returns an array of events. The final model response is the last `content.role == "model"` event with a `text` part.

```python
async def run_agent_session(
    agent_url: str, message: str, user_id: str, app_name: str, session_id: str
) -> tuple[str, str]:
    """Returns (response_text, effective_session_id)."""

    async def _post_run(_session_id: str):
        async with httpx.AsyncClient(timeout=60.0) as client:
            return await client.post(
                f"{agent_url}/run",
                headers={"Content-Type": "application/json"},
                json={
                    "app_name": app_name,
                    "user_id": user_id,
                    "session_id": _session_id,
                    "new_message": {"role": "user", "parts": [{"text": message}]},
                },
            )

    r = await _post_run(session_id)

    # Session expired / missing → create fresh and retry once
    if r.status_code == 404:
        new_session_id = await create_session(agent_url, user_id, app_name)
        r = await _post_run(new_session_id)
        r.raise_for_status()
        return _parse_events(r.json()), new_session_id

    r.raise_for_status()
    return _parse_events(r.json()), session_id


def _parse_events(events: list) -> str:
    """Extract final model text from ADK events array."""
    for event in reversed(events):
        content = event.get("content")
        if content and content.get("role") == "model" and isinstance(content.get("parts"), list):
            for part in reversed(content["parts"]):
                if isinstance(part, dict) and "text" in part:
                    return part["text"]
    return "Agent did not provide a final text response."
```

**ADK `/run` request format:**
```json
{
  "app_name": "jarvis_agent",
  "user_id": "user-tony-stark",
  "session_id": "session-1729354800",
  "new_message": {
    "role": "user",
    "parts": [{"text": "Hello!"}]
  }
}
```

---

## Pattern 4: Session 404 Auto-Recovery

The retry-once pattern handles expired or missing sessions gracefully. The caller gets back the new session_id so they can persist it.

```python
# In run_agent endpoint:
session_id = request.session_id or await create_session(...)
response_text, effective_session_id = await run_agent_session(..., session_id)

# effective_session_id may differ from session_id if a 404 triggered new session creation
return AgentResponse(session_id=effective_session_id, ...)
```

Rule: Always return `effective_session_id` (not the input one), so frontend always has a valid session.

---

## Pattern 5: History Fetch + Normalization

```python
class HistoryRequest(BaseModel):
    agent_name: str
    user_id: str
    session_id: str

@app.post("/get_history")
async def get_history(req: HistoryRequest):
    if not req.session_id:
        return {"history": []}

    agent_url = AGENT_REGISTRY[req.agent_name]
    url = f"{agent_url}/apps/{req.agent_name}/users/{req.user_id}/sessions/{req.session_id}"

    async with httpx.AsyncClient(timeout=30.0) as client:
        r = await client.get(url)
        if r.status_code == 404:
            return {"history": []}
        r.raise_for_status()
        return {"history": _normalize_events(r.json())}


def _normalize_events(session_json: dict) -> list:
    """Convert ADK session events → clean role/content list."""
    history = []
    for event in session_json.get("events", []):
        author = event.get("author")
        if author not in ("USER", "MODEL"):
            continue
        parts = event.get("content", {}).get("parts", [])
        if not parts:
            continue
        text = parts[0].get("text", "") if isinstance(parts[0], dict) else ""
        if not text:
            continue
        history.append({
            "role": "user" if author == "USER" else "assistant",
            "content": text
        })
    return history
```

**ADK session fetch URL format:**
```
GET {adk_base_url}/apps/{app_name}/users/{user_id}/sessions/{session_id}
```

Returns session JSON with `events` array containing full conversation history.

---

## Pattern 6: config.json + config.py — Environment-Aware Registry

**config.json:**
```json
{
  "environments": {
    "local": {
      "adk_bundle_url": "http://localhost:8000"
    },
    "cloud": {
      "adk_bundle_url": "https://your-adk-bundle.run.app"
    }
  },
  "agents": [
    "greeting_agent",
    "jarvis_agent",
    "calc_agent"
  ]
}
```

**config.py:**
```python
import json, os, logging
from typing import Dict

def load_config() -> Dict[str, str]:
    env = os.getenv("APP_ENV", "local")
    try:
        with open("config.json", "r") as f:
            config_data = json.load(f)
    except (FileNotFoundError, json.JSONDecodeError) as e:
        logging.error(f"Could not load config.json: {e}")
        return {}

    base_url = config_data.get("environments", {}).get(env, {}).get("adk_bundle_url")
    if not base_url:
        logging.warning(f"ADK bundle URL not found for environment '{env}'.")
        return {}

    agent_names = config_data.get("agents", [])
    return {name: base_url for name in agent_names}

AGENT_REGISTRY = load_config()
```

Key insight: **all agents share one base URL**. The registry is `{agent_name → base_url}` and the bundle URL is the same for all agents. Agent names are used as `app_name` in ADK API calls.

---

## Pattern 7: Utility Endpoints

```python
@app.get("/health")
async def health_check():
    return {"status": "healthy", "agents": list(AGENT_REGISTRY.keys())}

@app.get("/agents")
async def list_agents():
    return {"agents": list(AGENT_REGISTRY.keys())}
```

Always include `/health`. Useful for Cloud Run health checks and debugging.

---

## Pattern 8: APP_ENV Toggle — Local vs Cloud

```python
# Default: targets local ADK bundle
# Override to target live Cloud Run ADK bundle
APP_ENV=cloud uvicorn main:app --reload --port=8080
```

In `start_cloud.sh`:
```bash
APP_ENV="cloud" uvicorn main:app --reload --port=8080
```

In `Dockerfile`:
```dockerfile
ENV APP_ENV="cloud"
CMD uvicorn main:app --host 0.0.0.0 --port ${PORT}
```

Three modes of operation:
1. `APP_ENV=local` (default) — local ADK bundle on localhost:8000
2. `APP_ENV=cloud` + local run — targets live Cloud Run ADK bundle (for integration testing)
3. `APP_ENV=cloud` inside container — production

---

## Pattern 9: Minimal Wrapper Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV APP_ENV="cloud"

# Shell form (not exec form) so $PORT is substituted at runtime
CMD uvicorn main:app --host 0.0.0.0 --port ${PORT}
```

Important: Shell form CMD (`CMD uvicorn ...`) not exec form (`CMD ["uvicorn", ...]`). Only shell form performs `$PORT` env var substitution.

---

## Pattern 10: Minimal Wrapper deploy.sh

```bash
#!/bin/bash
set -e

SERVICE_NAME="adk-wrapper-prod-v2"
ADK_BUNDLE_URL="https://your-adk-bundle.run.app"

gcloud run deploy "$SERVICE_NAME" \
  --source . \
  --set-env-vars "ADK_BUNDLE_URL=${ADK_BUNDLE_URL}" \
  --allow-unauthenticated
```

The wrapper is simpler than the ADK bundle deploy:
- No `--set-secrets` (ADK bundle URL is not a secret)
- No `grant_permissions.sh` (no Secret Manager access needed)
- No `store_secrets.sh` (no secrets at all)

---

## Pattern 11: Frontend Client Code (Python)

How to call the wrapper from Streamlit or any Python frontend:

```python
import requests

WRAPPER_URL = "https://your-wrapper.run.app"

def ask_agent(agent_name: str, message: str, user_id: str, session_id: str = None) -> dict:
    payload = {
        "agent_name": agent_name,
        "message": message,
        "user_id": user_id,
        "session_id": session_id
    }
    response = requests.post(f"{WRAPPER_URL}/run_agent", json=payload, timeout=90)
    response.raise_for_status()
    return response.json()
    # Returns: {"response": "...", "session_id": "...", "agent_name": "...", "status": "success"}

def get_history(agent_name: str, user_id: str, session_id: str) -> list:
    payload = {"agent_name": agent_name, "user_id": user_id, "session_id": session_id}
    response = requests.post(f"{WRAPPER_URL}/get_history", json=payload, timeout=30)
    response.raise_for_status()
    return response.json().get("history", [])
    # Returns: [{"role": "user"|"assistant", "content": "..."}]
```

Session management in frontend:
```python
# First message — no session
result = ask_agent("jarvis_agent", "Hello!", "user-123")
session_id = result["session_id"]  # Save this!

# All subsequent messages — pass session_id back
result = ask_agent("jarvis_agent", "Tell me more.", "user-123", session_id=session_id)
session_id = result["session_id"]  # Always update — may change if 404 recovery triggered
```

---

## Pattern 12: Session ID Naming Convention

```python
session_id = f"session-{int(time.time())}"
# Example: "session-1729354800"
```

- Epoch timestamp = unique enough for typical usage
- Human readable — easy to debug in logs
- No UUID dependency needed
- Consistent across repos (adk-exp-v2, n8n-hybrid, this repo all use similar patterns)
