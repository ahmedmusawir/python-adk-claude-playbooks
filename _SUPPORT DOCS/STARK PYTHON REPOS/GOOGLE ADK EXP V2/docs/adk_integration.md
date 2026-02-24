# ADK Integration Deep-Dive: google-adk-exp-v2

## What is ADK?

Google Agent Development Kit (`google-adk`) — a Python framework for building multi-agent AI applications on Gemini. It provides:
- Agent primitives (Agent, SequentialAgent, ParallelAgent)
- Built-in tools (google_search, code execution, MCP)
- Session/context management (passes data between agents)
- A web UI (`adk web`) for interactive testing
- Cloud Run deployment via `adk deploy cloud_run`

**Version in this repo:** `google-adk==1.13.0`
**Requires:** Python 3.11+

---

## Installation & Setup

```bash
pip install google-adk

# For web search tool
# google_search is built-in, no extra install

# For GCS instructions (optional)
pip install google-cloud-storage

# For non-Gemini models
pip install litellm

# For MCP tools
pip install mcp
```

**Environment:**
```bash
# .env file
GOOGLE_GENAI_USE_VERTEXAI=FALSE   # Use Gemini API directly (not Vertex AI)
GOOGLE_API_KEY=your_key_here       # Get from https://ai.google.dev
```

---

## ADK Agent Types Reference

### `Agent` (LLM Agent)
- Has a model (LLM)
- Has tools it can call
- Has instruction (system prompt)
- Produces output

```python
from google.adk.agents import Agent

agent = Agent(
    name="str",                    # required; underscore format
    model="gemini-2.5-flash",      # required; string or LiteLlm instance
    description="str",             # required; tells orchestrators what this does
    instruction="str or callable", # required; system prompt
    tools=[],                      # optional; list of Tool instances
    output_key="str",              # optional; saves output to session context
    code_executor=None,            # optional; BuiltInCodeExecutor instance
)
```

### `SequentialAgent` (Orchestrator, No LLM)
- Runs sub-agents in order
- No model, no instruction
- Passes session context between stages

```python
from google.adk.agents import SequentialAgent

orchestrator = SequentialAgent(
    name="str",
    description="str",
    sub_agents=[agent_1, agent_2, agent_3],  # executed in this order
)
```

### `ParallelAgent` (Orchestrator, No LLM)
- Runs sub-agents concurrently
- No model, no instruction
- All sub-agent outputs go to session context simultaneously

```python
from google.adk.agents import ParallelAgent

parallel = ParallelAgent(
    name="str",
    description="str",
    sub_agents=[branch_1, branch_2, branch_3],  # all run at same time
)
```

---

## ADK Tools Reference

### Built-in Tools

```python
from google.adk.tools import google_search
# Use: tools=[google_search]
# Effect: agent can search the web

from google.adk.code_executors import BuiltInCodeExecutor
# Use: code_executor=BuiltInCodeExecutor()
# Effect: agent can write and execute Python code
# IMPORTANT: goes in code_executor= not tools=

# Legacy import (pre-1.0 style, still works):
from google.adk.tools import built_in_code_execution
# Use: tools=[built_in_code_execution]
```

### FunctionTool — Wrap Python Functions

```python
from google.adk.tools import FunctionTool

def my_function(param1: str, param2: int = 10) -> str:
    """
    Short description of what this tool does.         ← LLM reads this

    Args:
        param1: Description of param1.
        param2: Description of param2 (default: 10).

    Returns:
        Description of return value.
    """
    return "result"

tool = FunctionTool(func=my_function)
# Use: tools=[tool]
```

**Rule:** Always write complete docstrings. ADK uses them to generate tool descriptions for the LLM.

### AgentTool — Wrap Agent as Tool

```python
from google.adk.tools import AgentTool

tool = AgentTool(agent=my_agent_or_workflow)
# Use: tools=[tool]
# Effect: root agent can invoke `my_agent_or_workflow` as a tool
```

**Use case:** Chatbot root agent that delegates to complex workflows on demand.

### MCPToolset — Connect to MCP Servers

```python
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, StreamableHTTPConnectionParams

toolset = MCPToolset(
    connection_params=StreamableHTTPConnectionParams(
        url="https://mcp-server.example.com/mcp/",
        headers={
            "Authorization": "Bearer TOKEN",
            "Custom-Header": "value",
        }
    ),
    tool_filter=["tool_name_1", "tool_name_2"],  # optional whitelist
)
# Use: tools=[toolset]
```

---

## Session Context and `output_key`

The session context is ADK's key-value store for passing data between agents.

**Writing to context:**
```python
agent = Agent(
    name="my_agent",
    ...
    output_key="my_output",  # agent's final response → stored at this key
)
```

**Reading from context (in templates):**
```python
Agent(
    instruction="""
    Previous result: {my_output}
    Now do the next step.
    """,
)
# ADK substitutes {my_output} with the session context value
```

**Rules:**
- `output_key` must be unique across agents in the same pipeline
- Template variable names must exactly match `output_key` values
- Only string substitution — no complex expressions in templates

---

## Instruction Patterns

### Static String (Inline)
```python
agent = Agent(
    instruction="You are a helpful assistant. Always be polite.",
)
```
Best for: simple agents where the instruction never changes.

### Static String (Multiline)
```python
agent = Agent(
    instruction="""
    You are an expert at {domain}.

    When the user asks about {topic}, use the search tool.
    Always cite your sources.
    """,
)
```

### Callable (Hot-Reloadable)
```python
def get_instructions(ctx) -> str:
    return fetch_from_gcs()  # or database, or anywhere

agent = Agent(
    instruction=get_instructions,  # pass function object, NOT result
)
```
Best for: production agents where prompt tuning happens without redeployment.

---

## Running ADK Locally

```bash
# Interactive web UI (recommended for development)
adk web

# Run a specific agent in terminal
adk run greeting_agent

# Serve API only (headless)
adk api_server
```

**`adk web`** serves:
- Web chat UI at `localhost:8000`
- API at `localhost:8000/api/...`
- Select any agent from the repo

---

## Deployment to Cloud Run

### Method 1: via gcloud (using Dockerfile)
```bash
gcloud run deploy my-service-name \
  --source . \
  --allow-unauthenticated
```

### Method 2: via ADK CLI
```bash
adk deploy cloud_run \
  --project=YOUR_PROJECT_ID \
  --region=YOUR_REGION \
  --service_name=my-service-name \
  --with_ui \
  --port=8080 \
  ./my_agent
```

**Dockerfile for Cloud Run:**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["sh", "-c", "adk web --port $PORT"]
```
Key: use `$PORT` (Cloud Run injects this).

---

## LiteLlm Integration (Non-Gemini Models)

```python
from google.adk.models.lite_llm import LiteLlm
import os

# OpenRouter (access to GPT-4, Claude, etc.)
model = LiteLlm(
    model="openrouter/openai/gpt-4o",
    api_key=os.getenv("OPENROUTER_API_KEY"),
)

# OpenAI directly
model = LiteLlm(
    model="openai/gpt-4o",
    api_key=os.getenv("OPENAI_API_KEY"),
)

# Anthropic directly
model = LiteLlm(
    model="anthropic/claude-sonnet-4-5",
    api_key=os.getenv("ANTHROPIC_API_KEY"),
)

# Use in agent
agent = Agent(
    model=model,  # LiteLlm instance, not string
    ...
)
```

---

## Known Issues / Gotchas

### 1. `BuiltInCodeExecutor` goes in `code_executor=`, NOT `tools=[]`

```python
# WRONG:
agent = Agent(tools=[BuiltInCodeExecutor()])

# CORRECT:
agent = Agent(code_executor=BuiltInCodeExecutor())
```

### 2. Callable instruction vs. startup-cached instruction

```python
# CACHED (evaluated once at import time):
instruction=fetch_instructions("agent")

# FRESH (called on every agent run):
def get_instructions(ctx): return fetch_instructions("agent")
instruction=get_instructions
```

### 3. `SequentialAgent` and `ParallelAgent` have NO model

They are pure orchestrators. Do not pass `model=` to them.

### 4. `output_key` collision

If two agents in the same pipeline use the same `output_key`, the second overwrites the first. Always use unique keys.

### 5. ADK discovers `root_agent` via `__init__.py`

If `__init__.py` is missing or doesn't export `root_agent`, `adk web` won't see the agent.

### 6. LiteLlm model ID typo risk

Model IDs for OpenRouter must be: `"openrouter/provider/model-name"` (not `"openrcouter/..."`). The ghl_mcp_agent has a typo (`openrcouter` vs `openrouter`) that will cause runtime failures.

---

## N8N Integration

The repo includes `Adk N8N Hybrid v4.json` — an N8N workflow that connects to ADK agents via HTTP. Pattern:
- N8N workflow triggers (webhook, schedule, form)
- N8N HTTP node → ADK API server endpoint
- ADK processes via agent
- Response back to N8N for further automation

This is the "ADK as backend, N8N as orchestrator" pattern for non-technical workflow automation.

---

## Model Reference (As of 2026-02)

| Model ID | Speed | Quality | Cost | Use For |
|----------|-------|---------|------|---------|
| `gemini-2.5-flash` | Fast | High | Medium | Default for most agents |
| `gemini-2.0-flash` | Fast | Medium | Low | Simple sub-agents, retrieval |
| `gemini-1.5-flash` | Medium | Medium | Low | Legacy, prefer 2.0+ |
| `gemini-2.5-pro` | Slow | Highest | High | Complex reasoning, one-off |

Full model list: https://ai.google.dev/gemini-api/docs/models

---

## ADK vs Direct Vertex AI (Cross-Repo Comparison)

| Aspect | ADK (this repo) | Direct Vertex AI (VidGen) |
|--------|-----------------|---------------------------|
| Use case | Agent orchestration | Media generation pipeline |
| LLM calls | Via Agent framework | Direct API calls |
| Session mgmt | ADK handles it | Manual (file-based state) |
| Multi-agent | Native primitives | Not applicable |
| Tool calling | Native FunctionTool | Custom functions |
| Deployment | `adk deploy cloud_run` | Streamlit on Cloud Run |
| Auth | ADC or API key | ADC (service account) |

**Decision rule:** Use ADK for agent-first applications. Use direct Vertex AI SDK for media pipelines, batch processing, or when you need fine-grained control over model calls.

---

_Extracted: 2026-02-23_
_Repo: google-adk-exp-v2_
