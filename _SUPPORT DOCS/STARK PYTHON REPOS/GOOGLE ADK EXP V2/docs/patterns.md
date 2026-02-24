# Patterns: google-adk-exp-v2

Copy-pasteable ADK patterns extracted from working code.

---

## 1. Minimal ADK Agent (Baseline)

```python
from google.adk.agents import Agent

root_agent = Agent(
    name="my_agent",
    model="gemini-2.5-flash",
    description="What this agent does (used by orchestrators)",
    instruction="You are a helpful assistant. ...",
)
```

**Notes:**
- `name` — used internally by ADK; underscore format (`my_agent`)
- `model` — string model ID, defaults to Gemini
- `description` — shown to orchestrators deciding which agent to call
- `instruction` — system prompt, either a string or a callable
- `__init__.py` must export `root_agent`

---

## 2. Agent with Built-in Tools

```python
from google.adk.agents import Agent
from google.adk.code_executors import BuiltInCodeExecutor
from google.adk.tools import google_search

# Code execution tool
code_executor = BuiltInCodeExecutor()

agent = Agent(
    name="calc_agent",
    model="gemini-2.5-flash",
    description="Calculator agent",
    instruction="You can write and execute Python code to solve math problems.",
    code_executor=code_executor,  # code executor goes here, NOT in tools=[]
)

# Search tool (goes in tools=[])
search_agent = Agent(
    name="jarvis_agent",
    model="gemini-2.5-flash",
    description="Web-enabled assistant",
    instruction="Use google_search to find current information.",
    tools=[google_search],
)
```

**Critical:** `BuiltInCodeExecutor` goes in `code_executor=` parameter, NOT `tools=[]`.

---

## 3. FunctionTool — Wrap a Python Function

```python
from google.adk.tools import FunctionTool

def write_json_data(data: str, filename: str = "output.json"):
    """
    Writes data to a JSON file.            ← ADK reads this docstring for the LLM

    Args:
        data: The JSON string to write.
        filename: Output filename.

    Returns:
        Confirmation message string.
    """
    # ... implementation
    return f"Wrote to {file_path}"

# Wrap it
json_writer_tool = FunctionTool(func=write_json_data)

# Use in agent
agent = Agent(
    name="writer_agent",
    model="gemini-2.5-flash",
    description="Writes output to JSON",
    instruction="Use the json_writer tool to save data.",
    tools=[json_writer_tool],
)
```

**Notes:**
- ADK reads the docstring to describe the tool to the LLM — write good docstrings
- Type annotations on parameters help ADK generate correct function schemas

---

## 4. Sequential Agent Pipeline

```python
from google.adk.agents import Agent, SequentialAgent
from google.adk.tools import google_search

MODEL = "gemini-2.5-flash"

# Stage 1: produces output
stage_one = Agent(
    model=MODEL,
    name="IdeaAgent",
    description="Generates ideas based on user request.",
    instruction="Generate 3 ideas based on the user's request.",
    tools=[google_search],
    output_key="ideas",  # saves output to session context as "ideas"
)

# Stage 2: consumes stage 1 output via template variable
stage_two = Agent(
    model=MODEL,
    name="RefinerAgent",
    description="Filters ideas based on constraints.",
    instruction="""Review the provided ideas:
    Ideas: {ideas}              ← pulled from session context by key name
    Filter to only ideas that meet the budget constraint.
    """,
    output_key="refined_ideas",
)

# Orchestrator — runs stage_one then stage_two
root_agent = SequentialAgent(
    name="PlannerAgent",
    description="Orchestrates the planning workflow.",
    sub_agents=[stage_one, stage_two],
)
```

**Key rules:**
- `output_key="name"` saves the agent's output to session context under that key
- Use `{key_name}` in subsequent agents' instructions to access it
- `SequentialAgent` has no `model` — it's pure orchestration, no LLM

---

## 5. Parallel Agent (Concurrent Execution)

```python
from google.adk.agents import Agent, ParallelAgent
from google.adk.tools import google_search

MODEL = "gemini-2.0-flash"

branch_one = Agent(
    model=MODEL,
    name="NewsAgent",
    description="Finds latest news.",
    instruction="Use google_search to find recent news headlines for the given topic.",
    tools=[google_search],
    output_key="latest_news",
)

branch_two = Agent(
    model=MODEL,
    name="StockAgent",
    description="Finds current stock price.",
    instruction="Use google_search to find the current stock price.",
    tools=[google_search],
    output_key="current_stock_price",
)

# Both branches run concurrently
root_agent = ParallelAgent(
    name="ResearchAgent",
    description="Gathers news and stock prices simultaneously.",
    sub_agents=[branch_one, branch_two],
)
```

**Notes:**
- All sub-agents run concurrently (true parallelism)
- All `output_key` values become available in session context after parallel completes
- `ParallelAgent` has no `model` — pure orchestration

---

## 6. Combo Agent — Agent + AgentTool (Nesting)

```python
from google.adk.agents import Agent, ParallelAgent, SequentialAgent
from google.adk.tools import google_search, AgentTool

MODEL = "gemini-2.5-flash"

# Build the parallel sub-workflow
news_agent = Agent(model=MODEL, name="NewsAgent", ..., output_key="latest_news")
stock_agent = Agent(model=MODEL, name="StockAgent", ..., output_key="current_stock_price")

parallel_agent = ParallelAgent(
    name="ResearchAgent",
    sub_agents=[news_agent, stock_agent],
)

synthesis_agent = Agent(
    model=MODEL,
    name="SynthesisAgent",
    instruction="""
    News: {latest_news}
    Stock: {current_stock_price}
    Synthesize into a single response.
    """,
)

# Wrap the workflow as a SequentialAgent
research_workflow = SequentialAgent(
    name="ResearchWorkflow",
    sub_agents=[parallel_agent, synthesis_agent],
)

# Root agent uses the workflow as a tool
root_agent = Agent(
    model=MODEL,
    name="Chatbot",
    description="Friendly chatbot for news and stocks.",
    instruction="""You are a helpful chatbot.
    If the user asks for news or stock prices, use `research_tool`.
    For all other questions, respond conversationally.
    """,
    tools=[AgentTool(agent=research_workflow)],  # wraps the workflow as a callable tool
)
```

**When to use:** When you want a conversational front-end agent that delegates to complex workflows only when needed. Avoids running the full pipeline on every message.

---

## 7. Callable Instruction (Live/Hot-Reloadable)

```python
from google.adk.agents import Agent

def get_live_instructions(ctx) -> str:
    """Called by ADK on every agent run."""
    # Could fetch from GCS, database, or any source
    return fetch_from_somewhere()

root_agent = Agent(
    name="my_agent",
    model="gemini-2.5-flash",
    description="Agent with hot-reloadable instructions",
    instruction=get_live_instructions,  # pass function object, NOT function call
    # instruction=get_live_instructions()  ← WRONG: cached at startup
)
```

**Critical distinction:**
- `instruction=my_fn` — callable, called every run (hot-reloadable)
- `instruction=my_fn()` — string, evaluated once at startup (cached)

---

## 8. GCS Live Instructions Pattern

```python
# utils/gcs_utils.py
from google.cloud import storage

BUCKET_NAME = "your-bucket-name"
BASE_FOLDER = "ADK_Agent_Bundle_1"

def fetch_instructions(agent_name: str) -> str:
    """
    Fetches agent instructions from GCS.
    Path: {BASE_FOLDER}/{agent_name}/{agent_name}_instructions.txt
    """
    try:
        storage_client = storage.Client()
        bucket = storage_client.bucket(BUCKET_NAME)
        file_path = f"{BASE_FOLDER}/{agent_name}/{agent_name}_instructions.txt"
        blob = bucket.blob(file_path)
        return blob.download_as_text(encoding='utf-8')
    except Exception as e:
        print(f"ERROR: Could not fetch instructions for '{agent_name}'. Error: {e}")
        return f"Error: Could not load instructions for {agent_name}."

# In agent.py:
from utils.gcs_utils import fetch_instructions

def get_live_instructions(ctx) -> str:
    return fetch_instructions("my_agent")

root_agent = Agent(
    name="my_agent",
    model="gemini-2.5-flash",
    instruction=get_live_instructions,   # callable → fetched fresh every run
)
```

**GCS folder convention:**
```
BUCKET/
└── ADK_Agent_Bundle_1/
    ├── my_agent/
    │   └── my_agent_instructions.txt
    └── context_store/
        └── knowledge_doc.txt
```

---

## 9. GCS Context Store Tool (Agent reads arbitrary docs)

```python
# utils/context_utils.py
from google.cloud import storage
from google.cloud.exceptions import NotFound

BUCKET_NAME = "your-bucket-name"
FOLDER_NAME = "ADK_Agent_Bundle_1/context_store"

def fetch_document(file_name: str) -> str:
    """
    Reads the full text of a document from the knowledge base.

    Use this to answer questions about any document by passing the filename.

    Args:
        file_name: The name of the file (e.g., "resume.txt").

    Returns:
        The file content as a string, or an error message if not found.
    """
    try:
        storage_client = storage.Client()
        bucket = storage_client.bucket(BUCKET_NAME)
        blob = bucket.blob(f"{FOLDER_NAME}/{file_name}")
        return blob.download_as_text()
    except NotFound:
        return f"ERROR: Document '{file_name}' not found."
    except Exception as e:
        return f"An error occurred: {e}"

# In agent.py:
from google.adk.tools import FunctionTool
from utils.context_utils import fetch_document

doc_reader_tool = FunctionTool(func=fetch_document)

root_agent = Agent(
    name="my_agent",
    model="gemini-2.5-flash",
    instruction=get_live_instructions,
    tools=[doc_reader_tool],   # agent can now read any GCS doc on demand
)
```

---

## 10. LiteLlm — Non-Gemini Models in ADK

```python
import os
from google.adk.agents import Agent
from google.adk.models.lite_llm import LiteLlm

# Create the LiteLlm client
lite_llm_client = LiteLlm(
    model="openrouter/openai/gpt-4o",         # any OpenRouter model
    # model="openrouter/anthropic/claude-sonnet-4-5",
    api_key=os.getenv("OPENROUTER_API_KEY"),
)

root_agent = Agent(
    name="my_agent",
    model=lite_llm_client,   # pass client instance, not string
    description="Agent powered by OpenRouter",
    instruction="...",
)
```

**Supported via LiteLlm:**
- OpenAI models (gpt-4o, etc.)
- Anthropic models (claude-*)
- Any OpenRouter-proxied model
- Any other LiteLlm-supported provider

---

## 11. MCPToolset — Connect to MCP Servers

```python
import os
from google.adk.agents import Agent
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, StreamableHTTPConnectionParams

GHL_API_TOKEN = os.getenv("GHL_API_TOKEN")
GHL_LOCATION_ID = os.getenv("GHL_LOCATION_ID")

root_agent = Agent(
    name="ghl_mcp_agent",
    model=lite_llm_client,  # or string model
    description="CRM assistant",
    instruction="...",
    tools=[
        MCPToolset(
            connection_params=StreamableHTTPConnectionParams(
                url="https://services.leadconnectorhq.com/mcp/",
                headers={
                    "Authorization": f"Bearer {GHL_API_TOKEN}",
                    "locationId": GHL_LOCATION_ID,
                }
            ),
            # Optional: filter which MCP tools the agent can use
            # tool_filter=['contacts_get-contact', 'conversations_send-a-new-message']
        )
    ],
)
```

**Use for:** Any service that exposes an MCP endpoint. The agent automatically discovers all tools the MCP server provides. `tool_filter` limits available tools if the MCP server has many.

---

## 12. Agent Module Structure (Required)

Every agent folder needs exactly this structure:

```
my_agent/
├── agent.py       # defines root_agent
└── __init__.py    # exports root_agent
```

```python
# __init__.py (always exactly this)
from .agent import root_agent
```

ADK commands (`adk run`, `adk web`, `adk deploy`) discover `root_agent` via `__init__.py`.

---

## 13. Output Key Convention

```python
# Naming: use descriptive snake_case names
output_key="trip_ideas"          # ← stored in session context
output_key="refined_ideas"       # ← stored in session context
output_key="sami_opinion"        # ← stored in session context

# Template variables in subsequent agents:
instruction="""
Review the ideas: {trip_ideas}
Stock: {current_stock_price}
"""
# ADK substitutes {key_name} with the session context value
```

---

## 14. FunctionTool JSON Output Pattern

When an agent needs to write file output:

```python
# utils/output_utils.py
import json
from pathlib import Path

def write_json_data(data: str, filename: str = "results.json"):
    """
    Writes data to a JSON file in the output directory.

    Args:
        data: JSON string to write.
        filename: Output filename (default: results.json).

    Returns:
        Path to the written file as a string.
    """
    output_dir = Path("output/my_agent")
    output_dir.mkdir(parents=True, exist_ok=True)
    file_path = output_dir / filename

    try:
        data_to_write = json.loads(data)
    except json.JSONDecodeError:
        data_to_write = {"raw_output": data}  # graceful fallback

    with open(file_path, "w") as f:
        json.dump(data_to_write, f, indent=4)

    return f"Successfully wrote data to {file_path}"
```

**Pattern:** Always handle JSON decode errors gracefully. If the LLM returns slightly malformed JSON, wrap in `{"raw_output": data}` rather than crashing.

---

## 15. Dockerfile for Cloud Run

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["sh", "-c", "adk web --port $PORT"]
```

**Notes:**
- `$PORT` is injected by Cloud Run — always use this pattern
- `adk web` serves the ADK UI + API server
- `adk api_server` is the headless version (no UI) — use when you only need the API

---

_Extracted: 2026-02-23_
_Repo: google-adk-exp-v2_
