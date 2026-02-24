# ADK Agents: Fundamentals

## What This Is For

Read this to build any single-agent ADK app from scratch in under 30 minutes. Covers the complete agent lifecycle, the session amnesia fix, event loop reading, and the module structure ADK expects.

Assumes you know Python and have used an LLM API before. This starts from "here's the ADK-specific way," not "here's what a language model is."

---

## Quick Reference — Minimal Working Agent

Copy this, fill in your instruction, and run it.

```python
# my_agent/agent.py
import os
os.environ["GOOGLE_GENAI_USE_VERTEXAI"] = "1"        # Must be before ADK imports
os.environ["GOOGLE_CLOUD_PROJECT"] = "your-project-id"
os.environ["GOOGLE_CLOUD_LOCATION"] = "us-central1"

from google.adk.agents import Agent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService

# ─── GLOBAL SCOPE — required for session persistence ───────────────────────
session_service = InMemorySessionService()

root_agent = Agent(
    name="my_agent",
    model="gemini-2.5-flash",
    description="A helpful assistant.",
    instruction="You are a helpful assistant. Answer questions clearly and concisely.",
    tools=[],
)

runner = Runner(
    agent=root_agent,
    app_name="my_app",
    session_service=session_service,
)
```

```python
# my_agent/__init__.py
from .agent import root_agent
```

Run locally:
```bash
pip install google-adk
adk web              # browser playground at localhost:8000
adk api_server .     # FastAPI mode (preferred for production)
```

---

## Core Concepts

Think of ADK as a thin harness that: routes user messages to the right agent, manages conversation history in sessions, and streams back events (text chunks + tool calls). You provide the agent definition; ADK handles the scaffolding.

**The session is the conversation.** ADK uses a session service to store history. `InMemorySessionService` works for development and single-container deployments. For multi-instance Cloud Run, pass `--session_service_uri=postgresql://...` to `adk api_server`.

**The runner executes the agent.** One runner per app. Connects the session service to the agent, routes messages, streams back events.

---

## Agent Anatomy

```python
from google.adk.agents import Agent

root_agent = Agent(
    name="my_agent",          # Unique ID. No spaces. Used as routing key.
    model="gemini-2.5-flash", # Model ID. See model reference below.
    description="...",        # One sentence. Used by parent agents for routing decisions.
    instruction="...",        # System prompt. String or callable (for dynamic/GCS prompts).
    tools=[],                 # FunctionTool, AgentTool, MCPToolset, built-ins.
    output_key="result",      # Optional. Writes final text to session.state[output_key].
)
```

### Field Reference

| Field | Type | Notes |
|-------|------|-------|
| `name` | str | No spaces. Used as `agent_name` in API routing. |
| `model` | str | `gemini-2.5-flash` default. `gemini-2.5-pro` for complex reasoning. |
| `description` | str | One sentence. Parent agents read this to pick which sub-agent to call. |
| `instruction` | str \| Callable | If callable, called at request time — enables GCS-fetched dynamic prompts. |
| `tools` | list | Any combination of built-ins, FunctionTool, AgentTool, MCPToolset. |
| `output_key` | str | Writes agent's final response into `session.state[output_key]` for downstream agents. |

### Model Reference

| Model ID | Use For |
|----------|---------|
| `gemini-2.5-flash` | Default for everything. Fast, cheap, good quality. Standard pattern (10+ repos). |
| `gemini-2.5-pro` | Complex reasoning, nuanced instructions. ~5x more expensive. |
| `gemini-2.0-flash` | Legacy. Use 2.5 flash unless you have a specific reason. |

---

## The Amnesia Fix — Runner + SessionService at Module Scope

**Critical pattern. Appears in every single production repo.** If you create `InMemorySessionService()` inside a function or inside a Streamlit callback, ADK loses conversation history on every request. The session service must be a module-level global.

**Wrong — agent gets amnesia every request:**
```python
# app.py
def handle_message(user_input):
    session_service = InMemorySessionService()  # ❌ New instance every call = no memory
    runner = Runner(agent=root_agent, app_name="app", session_service=session_service)
    ...
```

**Right — session persists across requests:**
```python
# agent.py — top level, outside any function
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService

session_service = InMemorySessionService()  # ✅ Module-level global

runner = Runner(
    agent=root_agent,
    app_name="my_app",
    session_service=session_service,
)
```

Confirmed standard pattern across: GHL HTML Email Agent, ADK Hybrid V2, GHL MCP Agent, ADK EXP V2, ADK Streamlit V2 (all production repos).

---

## Environment Variables — Required

Set at the top of `agent.py`, **before any ADK imports**. The `google.adk` package reads these at import time.

```python
import os
os.environ["GOOGLE_GENAI_USE_VERTEXAI"] = "1"        # Switches SDK to use Vertex AI
os.environ["GOOGLE_CLOUD_PROJECT"] = "your-project-id"
os.environ["GOOGLE_CLOUD_LOCATION"] = "us-central1"

# Only after env vars are set:
from google.adk.agents import Agent
from google.adk.runners import Runner
```

In production (Cloud Run), inject via `gcloud run deploy --set-env-vars`. Don't hardcode in source. For local dev only:
```python
# Safe pattern: only set if not already in environment
import os
os.environ.setdefault("GOOGLE_GENAI_USE_VERTEXAI", "1")
os.environ.setdefault("GOOGLE_CLOUD_PROJECT", "your-project")
os.environ.setdefault("GOOGLE_CLOUD_LOCATION", "us-central1")
```

---

## get_session() Returns None — Always Check

`session_service.get_session()` returns `None` on a cache miss. It does NOT raise. If you use the return value without checking, you get a silent `NoneType` error downstream.

```python
APP_NAME = "my_app"
USER_ID = "default_user"

def get_or_create_session(session_id: str):
    session = session_service.get_session(
        app_name=APP_NAME,
        user_id=USER_ID,
        session_id=session_id,
    )
    if session is None:          # ← explicit check required
        session = session_service.create_session(
            app_name=APP_NAME,
            user_id=USER_ID,
            session_id=session_id,
        )
    return session
```

---

## asyncio.run() — The Sync Wrapper

ADK's runner is async. Streamlit and simple script contexts are sync. Use this wrapper:

```python
import asyncio
from google.genai import types

def run_agent_sync(user_input: str, session_id: str = "default") -> str:
    """Sync wrapper around async ADK runner. Safe for Streamlit and scripts."""
    async def _run():
        get_or_create_session(session_id)

        content = types.Content(
            role="user",
            parts=[types.Part.from_text(text=user_input)]
        )

        response_text = ""
        async for event in runner.run_async(
            user_id=USER_ID,
            session_id=session_id,
            new_message=content,
        ):
            if event.text:
                response_text += event.text

        return response_text

    return asyncio.run(_run())
```

**In Streamlit:** `asyncio.run()` works fine — Streamlit is synchronous, so there's no existing event loop conflict.

**In FastAPI async endpoints:** Don't use `asyncio.run()`. Use `await runner.run_async()` directly or run the sync wrapper in a thread pool.

---

## Event Loop — Reading Events

The runner yields events. Each event can be a text chunk, tool call, tool result, or lifecycle signal.

```python
async for event in runner.run_async(
    user_id=user_id,
    session_id=session_id,
    new_message=content,
):
    # Text response chunks
    if event.text:
        response_text += event.text

    # Tool calls starting (for status feedback in UI)
    if event.get_function_calls():
        for fn_call in event.get_function_calls():
            tool_name = fn_call.name
            tool_args = fn_call.args
            # Update UI: "Running: search_web..."

    # Tool results returning
    if event.get_function_responses():
        for fn_resp in event.get_function_responses():
            # Tool completed
            pass
```

### status_callback Pattern for Tool Feedback

Detect tool calls in the event loop and surface them to the UI:

```python
def handle_event(event, status_callback=None) -> str:
    """Process one event, return text if any. Call status_callback on tool calls."""
    if event.get_function_calls():
        for fn_call in event.get_function_calls():
            if status_callback:
                status_callback(fn_call.name)
            else:
                print(f"Tool: {fn_call.name}")
    if event.text:
        return event.text
    return ""
```

In Streamlit, pass `st.toast` as the callback:
```python
# Inside a Streamlit app:
handle_event(event, status_callback=lambda name: st.toast(f"⚙️ {name}", icon="🔧"))
```

---

## Agent Module Structure

ADK discovers agents by looking for `root_agent` in each subfolder's `__init__.py` when you run `adk web .` or `adk api_server .` from the project root.

```
my_project/
├── my_agent/
│   ├── __init__.py      # Exports root_agent — required by ADK discovery
│   ├── agent.py         # Agent + session_service + runner definitions
│   └── tools.py         # Optional: FunctionTool implementations
└── requirements.txt
```

**`__init__.py`** — exactly this, nothing more:
```python
from .agent import root_agent
```

The export name `root_agent` is not configurable. ADK's discovery mechanism looks for this exact name.

---

## adk web vs adk api_server

| Command | What It Starts | Use For |
|---------|---------------|---------|
| `adk web .` | Browser playground UI at :8000 | Dev testing, quick iteration |
| `adk api_server . --port 8000` | FastAPI server at :8000 | Production, Next.js/Streamlit integration |

`adk api_server` exposes the native ADK session API (create session → run agent → get events). This is what you deploy to Cloud Run. The port reads `$PORT` env var automatically.

In your Dockerfile, use **shell form CMD** (not JSON form) so `$PORT` gets substituted:
```dockerfile
# ✅ Shell form — $PORT is substituted
CMD adk api_server . --port $PORT

# ❌ JSON form — $PORT is literal string
CMD ["adk", "api_server", ".", "--port", "$PORT"]
```

---

## Built-In Tools

No implementation needed — import and add to `tools` list:

```python
from google.adk.tools import google_search, code_execution

root_agent = Agent(
    tools=[google_search],       # Real-time web search
    # tools=[code_execution],    # Python sandbox execution
)
```

---

## Complete Working Example

Search agent with conversation memory:

```python
# search_agent/agent.py
import os
os.environ["GOOGLE_GENAI_USE_VERTEXAI"] = "1"
os.environ["GOOGLE_CLOUD_PROJECT"] = "your-project"
os.environ["GOOGLE_CLOUD_LOCATION"] = "us-central1"

import asyncio
from google.adk.agents import Agent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.adk.tools import google_search
from google.genai import types

APP_NAME = "search_app"
USER_ID = "user_001"

session_service = InMemorySessionService()

root_agent = Agent(
    name="search_agent",
    model="gemini-2.5-flash",
    description="Answers questions using Google Search.",
    instruction="""You are a research assistant. Use Google Search to find accurate,
    up-to-date information. Always cite the source of your information.""",
    tools=[google_search],
)

runner = Runner(
    agent=root_agent,
    app_name=APP_NAME,
    session_service=session_service,
)


def chat(user_input: str, session_id: str = "default") -> str:
    async def _run():
        session = session_service.get_session(
            app_name=APP_NAME, user_id=USER_ID, session_id=session_id
        )
        if session is None:
            session_service.create_session(
                app_name=APP_NAME, user_id=USER_ID, session_id=session_id
            )

        content = types.Content(
            role="user",
            parts=[types.Part.from_text(text=user_input)]
        )

        response = ""
        async for event in runner.run_async(
            user_id=USER_ID,
            session_id=session_id,
            new_message=content,
        ):
            if event.text:
                response += event.text

        return response

    return asyncio.run(_run())


if __name__ == "__main__":
    print(chat("What are the latest developments in AI agents?"))
    print(chat("Tell me more about that last point."))  # Has memory of previous turn
```

```python
# search_agent/__init__.py
from .agent import root_agent
```

---

## Gotchas

**1. GOOGLE_GENAI_USE_VERTEXAI must be set before imports**
The SDK reads env vars at import time. Anything after `from google.adk.agents import Agent` is too late. Always put `os.environ` at the very top.

**2. get_session() returns None silently**
No exception on cache miss. Missing this check caused errors in multiple repos during initial development.

**3. InMemorySessionService must be module-level**
Streamlit reruns on every user interaction. A session service inside a function gets recreated on every rerun, destroying conversation history.

**4. root_agent is a required exact name**
ADK's discovery mechanism looks for a variable named `root_agent` in `__init__.py`. Any other name (like `agent` or `my_agent`) and `adk web` won't find it.

**5. asyncio.run() creates a new event loop**
Fine in Streamlit (sync). In a FastAPI async endpoint, use `await` on `runner.run_async()` instead — `asyncio.run()` will fail if an event loop is already running.

**6. Shell form CMD in Dockerfile**
JSON array CMD doesn't expand `$PORT`. Shell form does. Use `CMD adk api_server . --port $PORT`, not `CMD ["adk", "api_server", ".", "--port", "$PORT"]`.

---

## What's Next

- Multi-agent pipelines, AgentTool, MCPToolset → `adk-advanced-patterns.md`
- Expose as REST API for Next.js → `fastapi-adk-gateway.md`
- Build Streamlit UI → `streamlit-patterns.md`
- Deploy to Cloud Run → `cloud-run-deployment.md`
