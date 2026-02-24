# ADK Advanced Patterns

## What This Is For

Multi-agent pipelines, AgentTool (agent as a tool), MCPToolset for external tool servers, GCS-fetched dynamic instructions, LiteLlm for non-Vertex models, and the gotchas that break complex agent setups.

Read `adk-agents-fundamentals.md` first.

---

## Quick Reference — Agent Type Matrix

| Type | When To Use | ADK Class |
|------|------------|-----------|
| Single Agent | One task, one model | `Agent` |
| Sequential Agent | Pipeline: A → B → C in order | `SequentialAgent` |
| Parallel Agent | Fan-out: A, B, C all at once | `ParallelAgent` |
| Agent + AgentTool | Agent calls another agent as a tool | `Agent` + `AgentTool()` |
| LiteLlm Agent | Non-Vertex models (GPT, Claude, Mistral) | `Agent(model=LiteLlm(...))` |

---

## Pattern 1: Sequential Agent — Pipeline

Agents run in order. Each agent receives the conversation history from the previous. Use `output_key` to pass data between agents.

```python
from google.adk.agents import Agent, SequentialAgent
from google.adk.tools import google_search

# Stage 1: Research
research_agent = Agent(
    name="research_agent",
    model="gemini-2.5-flash",
    description="Researches a topic using web search.",
    instruction="Search the web for information about the given topic. Be thorough.",
    tools=[google_search],
    output_key="research_results",   # Saves output to session.state["research_results"]
)

# Stage 2: Writer (reads research_results from session state)
writer_agent = Agent(
    name="writer_agent",
    model="gemini-2.5-flash",
    description="Writes a report based on research.",
    instruction="""Write a clear, structured report based on the research findings.
    The research results are available in the session context.""",
    output_key="final_report",
)

# Pipeline: research → write
pipeline = SequentialAgent(
    name="research_pipeline",
    description="Researches a topic and writes a report.",
    sub_agents=[research_agent, writer_agent],
)

# This is what gets exported as root_agent
root_agent = pipeline
```

**output_key behavior:** After an agent completes, ADK writes its final text response into `session.state[output_key]`. The next agent in the pipeline can reference this in its instruction or via a tool that reads `session.state`.

---

## Pattern 2: Parallel Agent — Fan-Out

Agents run concurrently. Use when tasks are independent. Results collected after all complete.

```python
from google.adk.agents import Agent, ParallelAgent

# Run these three simultaneously
sentiment_agent = Agent(
    name="sentiment_agent",
    model="gemini-2.5-flash",
    description="Analyzes sentiment of text.",
    instruction="Analyze the sentiment of the provided text. Return: positive/negative/neutral with confidence score.",
    output_key="sentiment",
)

summary_agent = Agent(
    name="summary_agent",
    model="gemini-2.5-flash",
    description="Summarizes text.",
    instruction="Provide a concise 2-3 sentence summary of the provided text.",
    output_key="summary",
)

keywords_agent = Agent(
    name="keywords_agent",
    model="gemini-2.5-flash",
    description="Extracts keywords.",
    instruction="Extract the 5 most important keywords from the text.",
    output_key="keywords",
)

analyzer = ParallelAgent(
    name="text_analyzer",
    description="Analyzes text for sentiment, summary, and keywords simultaneously.",
    sub_agents=[sentiment_agent, summary_agent, keywords_agent],
)
```

---

## Pattern 3: Combo Agent — Sequential + Parallel Mixed

Sequential pipeline where one stage is a parallel fan-out.

```python
from google.adk.agents import Agent, SequentialAgent, ParallelAgent

# Stage 1: Fetch data
fetcher = Agent(
    name="data_fetcher",
    model="gemini-2.5-flash",
    instruction="Fetch and organize the requested data.",
    output_key="raw_data",
)

# Stage 2: Parallel analysis
analyzer = ParallelAgent(
    name="analyzer",
    sub_agents=[sentiment_agent, summary_agent, keywords_agent],
)

# Stage 3: Synthesize
synthesizer = Agent(
    name="synthesizer",
    model="gemini-2.5-flash",
    instruction="Synthesize the analysis results into a final report.",
)

# The pipeline
root_agent = SequentialAgent(
    name="full_pipeline",
    sub_agents=[fetcher, analyzer, synthesizer],
)
```

---

## Pattern 4: AgentTool — Agent as a Tool (Critical Gotcha)

When you need an agent to call another agent as a tool (not in a pipeline), use `AgentTool`. This is the fix for a specific ADK limitation: **you cannot mix Grounding tools (like `google_search`) with `FunctionTool` in the same agent**. ADK throws `400 INVALID_ARGUMENT`.

**The fix:** Wrap the Grounding-tool agent inside `AgentTool`, then use the `AgentTool` as a tool in the parent agent.

```python
from google.adk.agents import Agent
from google.adk.tools import AgentTool, FunctionTool, google_search

# Agent that only uses Grounding tools
search_specialist = Agent(
    name="search_specialist",
    model="gemini-2.5-flash",
    description="Performs web searches.",
    instruction="Search the web for the given query and return the findings.",
    tools=[google_search],  # Grounding tool — cannot mix with FunctionTool
)

# A regular FunctionTool — can coexist with AgentTool but NOT with google_search directly
def analyze_text(text: str, focus: str = "general") -> dict:
    """Analyze text and return structured insights."""
    return {"input_length": len(text), "focus": focus, "status": "analyzed"}

# Parent agent uses AgentTool to call the search specialist
root_agent = Agent(
    name="orchestrator",
    model="gemini-2.5-flash",
    description="Orchestrates research and analysis tasks.",
    instruction="""You are a research orchestrator. Use the search_specialist tool
    to search for information, then analyze and summarize the results.""",
    tools=[
        AgentTool(agent=search_specialist),  # ← wraps agent as a callable tool
        FunctionTool(func=analyze_text),     # ← FunctionTool safely alongside AgentTool
    ],
)
```

**Why this works:** `AgentTool` runs the sub-agent in a sub-session, shielding the parent from the Grounding tool conflict. The parent agent sees it as a regular function call.

Confirmed from: GHL HTML Email Template Generator. Caused `400 INVALID_ARGUMENT` before fix was found.

---

## Pattern 5: MCPToolset — External MCP Server

Connect an ADK agent to an MCP server (TypeScript or Python). The agent gains access to all the MCP server's tools.

```python
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, StreamableHTTPConnectionParams

# Full MCPToolset — all tools from the MCP server
mcp_tools = MCPToolset(
    connection_params=StreamableHTTPConnectionParams(
        url="http://localhost:9000/mcp",   # MCP server URL
    )
)

root_agent = Agent(
    name="crm_agent",
    model="gemini-2.5-flash",
    description="Interacts with GHL CRM.",
    instruction="""You are a CRM assistant. Use the available MCP tools to:
    - Search for contacts
    - View conversations
    - Update contact information
    Always execute one tool at a time and wait for results before the next call.""",
    tools=[mcp_tools],
)
```

### tool_filter — Selective Tool Exposure

Expose only specific tools from the MCP server (prevents the agent from seeing all 200+ tools):

```python
mcp_tools = MCPToolset(
    connection_params=StreamableHTTPConnectionParams(
        url="http://localhost:9000/mcp",
    ),
    tool_filter=["search_contacts", "get_contact", "update_contact", "get_conversations"]
)
```

`tool_filter` is a list of tool names (as registered in the MCP server). The agent only sees these tools.

### LiteLlm + MCPToolset

For agents using non-Vertex models with MCPToolset:

```python
from google.adk.models.lite_llm import LiteLlm

mcp_tools = MCPToolset(
    connection_params=StreamableHTTPConnectionParams(url="http://localhost:9000/mcp"),
)

root_agent = Agent(
    name="gpt_crm_agent",
    model=LiteLlm(model="openai/gpt-4o"),
    description="CRM assistant powered by GPT-4o.",
    instruction="You are a CRM assistant. Use the available tools to search contacts and manage conversations. Execute tools one at a time.",
    tools=[mcp_tools],
)
```

---

## Pattern 6: output_key — Inter-Agent Data Flow

`output_key` writes the agent's final response text to `session.state`. Use in sequential pipelines to pass data between agents.

```python
stage_1 = Agent(
    name="extractor",
    model="gemini-2.5-flash",
    description="Extracts key data points from user input.",
    instruction="Extract all key data points from the user's input. Return them in a structured format.",
    output_key="extracted_data",    # session.state["extracted_data"] = agent's response
)

stage_2 = Agent(
    name="analyzer",
    model="gemini-2.5-flash",
    description="Analyzes extracted data and produces insights.",
    instruction="""Analyze the extracted data from the previous stage.
    The data is in your conversation context.""",
    output_key="analysis",
)
```

To read `session.state` from a tool:
```python
from google.adk.agents.callback_context import CallbackContext

def my_tool(context: CallbackContext, input: str) -> dict:
    # Read data from session state
    previous_output = context.state.get("extracted_data", "")
    # ...
```

---

## Pattern 7: GCS Callable Instructions (Hot-Reload)

Fetch system prompts from GCS at request time. Edit prompts in GCS → agent uses new prompt immediately. No redeploy.

```python
from google.cloud import storage

def get_agent_instruction() -> str:
    """Fetch current instruction from GCS. Called on every agent request."""
    client = storage.Client()
    bucket = client.bucket("your-prompts-bucket")
    blob = bucket.blob("prompts/crm_agent.md")
    return blob.download_as_text()

root_agent = Agent(
    name="crm_agent",
    model="gemini-2.5-flash",
    description="CRM assistant with hot-reloadable prompts.",
    instruction=get_agent_instruction,   # ← callable, NOT get_agent_instruction()
    tools=[],
)
```

**Critical gotcha:** `instruction=fn` passes the function. `instruction=fn()` calls it once at import time and stores the result as a static string. Use `fn`, not `fn()`.

For lower latency, cache with a short TTL:
```python
import time

_prompt_cache = {"text": None, "loaded_at": 0}
CACHE_TTL = 60  # seconds

def get_agent_instruction() -> str:
    now = time.time()
    if _prompt_cache["text"] is None or now - _prompt_cache["loaded_at"] > CACHE_TTL:
        client = storage.Client()
        _prompt_cache["text"] = client.bucket("prompts-bucket").blob("my_agent.md").download_as_text()
        _prompt_cache["loaded_at"] = now
    return _prompt_cache["text"]
```

---

## Pattern 8: LiteLlm — Non-Vertex Models

Use any model supported by LiteLLM (OpenAI, Anthropic, Mistral, etc.) with ADK agents.

```python
from google.adk.agents import Agent
from google.adk.models.lite_llm import LiteLlm
import os

# OpenAI
gpt_agent = Agent(
    name="gpt_agent",
    model=LiteLlm(model="openai/gpt-4o"),
    description="General assistant powered by GPT-4o.",
    instruction="You are a helpful assistant. Answer questions clearly and concisely.",
)

# Anthropic
claude_agent = Agent(
    name="claude_agent",
    model=LiteLlm(model="anthropic/claude-3-5-sonnet-20241022"),
    description="General assistant powered by Claude.",
    instruction="You are a helpful assistant. Answer questions clearly and concisely.",
)

# Env vars needed:
# OPENAI_API_KEY for OpenAI
# ANTHROPIC_API_KEY for Anthropic
```

LiteLlm agents work with `FunctionTool` but not with ADK's built-in Grounding tools (`google_search`). For search with LiteLlm, use a FunctionTool wrapper around a search API.

---

## Pattern 9: Sequential Tool Execution Instruction

**Critical for MCP agents.** ADK may try to call multiple MCP tools in parallel. This creates race conditions in the MCP server's session state (session poisoning). Force sequential execution via instruction.

```python
root_agent = Agent(
    name="crm_agent",
    instruction="""You are a CRM assistant with access to GHL tools.

    CRITICAL: Always execute tools ONE AT A TIME. Never call multiple tools simultaneously.
    Wait for each tool's result before making the next tool call.
    This prevents data corruption in the CRM system.

    Workflow:
    1. Call first tool
    2. Wait for result
    3. Analyze result
    4. Call next tool if needed
    5. Repeat until the user's request is fully handled
    """,
    tools=[mcp_tools],
)
```

First seen in: GHL Agent with MCP Server. The MCP server was getting corrupted sessions from parallel tool calls.

---

## Pattern 10: Global In-Module Tool State

When tools need to share state (e.g., an HTML editor where multiple tools read/modify the same document), use module-level globals as a bridge.

```python
# tools.py

_active_html: str = ""   # Module-level global — shared across all tool calls

def get_current_html() -> dict:
    """Tool: Returns current HTML content."""
    return {"html": _active_html}

def set_html_content(html: str) -> dict:
    """Tool: Sets the HTML content."""
    global _active_html
    _active_html = html
    return {"status": "set", "length": len(html)}

def apply_edit(selector: str, css_property: str, value: str) -> dict:
    """Tool: Applies CSS edit to current HTML."""
    global _active_html
    from bs4 import BeautifulSoup
    soup = BeautifulSoup(_active_html, "html.parser")
    elements = soup.select(selector)
    for el in elements:
        existing_style = el.get("style", "")
        el["style"] = f"{existing_style}; {css_property}: {value}".strip("; ")
    _active_html = str(soup)
    return {"status": "applied", "elements_modified": len(elements)}
```

In `agent.py`:
```python
from tools import get_current_html, set_html_content, apply_edit
from google.adk.tools import FunctionTool

root_agent = Agent(
    tools=[
        FunctionTool(func=get_current_html),
        FunctionTool(func=set_html_content),
        FunctionTool(func=apply_edit),
    ],
)
```

**Why this works:** ADK tools are stateless functions, but module globals persist across calls within a single process. The HTML state persists between tool calls in the same session.

Confirmed in: GHL HTML Email Template Generator — the only production pattern for document-editing agents.

---

## Pattern 11: Named Agent Protocols in System Prompt

For complex multi-step agent behaviors (human-in-the-loop, error recovery), define named protocols in the system prompt.

```python
instruction = """
You are a CRM assistant. Follow these protocols:

## Protocol Phoenix (Error Recovery)
If a tool returns an error or unexpected result:
1. Report the specific error to the user
2. Ask if they want to retry or try a different approach
3. Do NOT attempt automatic recovery without user confirmation

## Protocol 10 (Human-in-the-Loop for Destructive Actions)
Before any action that modifies or deletes data:
1. State exactly what you're about to do
2. List the specific records/data that will be affected
3. Ask for explicit confirmation with "Please type YES to confirm"
4. Only proceed after receiving YES

NEVER bypass Protocol 10 for any reason.
"""
```

These named protocols give the agent a vocabulary for self-regulation. Other agents or users can invoke them by name: "Use Protocol Phoenix on that error."

---

## Gotchas

**1. Cannot mix Grounding tools + FunctionTool**
`google_search` is a Grounding tool. You can't put it in the same agent as a `FunctionTool`. ADK throws `400 INVALID_ARGUMENT`. Fix: wrap the Grounding agent in `AgentTool` and use that in the parent agent.

**2. instruction=fn not instruction=fn()**
`instruction=get_instruction` passes the function (called on each request).
`instruction=get_instruction()` calls it once at import time. The agent will use the same prompt forever and ignore GCS updates.

**3. MCPToolset parallel calls poison sessions**
The MCP SDK's session management doesn't handle parallel calls from the same agent. Concurrent tool calls corrupt the session state. Enforce sequential execution via system prompt instruction.

**4. output_key is string-only**
`output_key` stores the agent's final text response as a string. If you need structured data between agents, have the agent output JSON and parse it in the next agent's tool.

**5. LiteLlm can't use ADK built-in tools**
`google_search` and `code_execution` are Vertex AI integrations. LiteLlm agents can't access them. Use FunctionTool wrappers around equivalent APIs instead.

**6. SequentialAgent passes context, not output_key**
The next agent in a sequence sees the conversation history (including previous agent responses), but doesn't automatically get `session.state[output_key]` injected into its context. Use a tool or an instruction that tells the agent to look for the previous agent's output in the conversation.

---

## What's Next

- Connect to TypeScript MCP server → `mcp-server-typescript.md`
- Use Vertex AI modalities (Imagen, TTS, STT) → `vertex-ai-integrations.md`
- Deploy multi-agent bundle to Cloud Run → `cloud-run-deployment.md`
