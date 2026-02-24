# Architecture: google-adk-exp-v2

## What This Repo Is

An ADK (Google Agent Development Kit) experimentation and reference repo. Not a single app — a **collection of working agent examples** demonstrating every major ADK pattern:

- Simple single-LLM agents
- Agents with built-in tools (code execution, web search)
- Sequential multi-agent pipelines
- Parallel multi-agent workflows
- Nested/combo agent topologies
- MCP-connected agents (external services)
- Live instructions from GCS (hot-reloadable prompts)
- Non-Gemini models via LiteLlm

**Value:** Use as a reference library when building any ADK-based backend.

---

## Repo File Structure

```
google-adk-exp-v2/
├── greeting_agent/          # Baseline: simplest possible ADK agent
│   ├── agent.py             # Current version (inline instruction)
│   ├── agent_gcs.py         # GCS callable instruction + FunctionTool
│   ├── agent.gc_bucket.py   # GCS instruction fetched at startup (cached)
│   ├── agent.org.py         # Original version
│   └── __init__.py          # from .agent import root_agent
│
├── calc_agent/              # Agent with code execution tool
│   ├── agent.py             # Uses BuiltInCodeExecutor + GCS callable instruction
│   ├── agent.org.py         # Original with tools=[built_in_code_execution]
│   └── __init__.py
│
├── jarvis_agent/            # Agent with google_search + GCS callable instruction
│   ├── agent.py             # Inline instruction version
│   ├── agent_gcs.py         # GCS callable instruction version
│   └── __init__.py
│
├── travel_agent/            # SequentialAgent: IdeaAgent → RefinerAgent
│   ├── agent.py             # output_key + template variable {trip_ideas}
│   └── __init__.py
│
├── parallel_agent/          # ParallelAgent: NewsAgent + StockAgent concurrently
│   ├── agent.py
│   └── __init__.py
│
├── combo_agent/             # Complex nesting: Agent(AgentTool(Sequential(Parallel, Synth)))
│   ├── agent.py
│   └── __init__.py
│
├── focus_group_agent/       # Sequential(Parallel(7 personas), JsonWriterAgent)
│   ├── agent.py             # Parallel focus group + file output via FunctionTool
│   └── __init__.py
│
├── ghl_mcp_agent/           # MCP agent connecting to GoHighLevel CRM
│   ├── agent.py             # LiteLlm + MCPToolset + StreamableHTTPConnectionParams
│   ├── agent_gcs.py         # GCS instruction variant
│   ├── agent.gc_bucket.py   # Startup-cached GCS variant
│   ├── agent.org.py         # Original (greeting agent scaffold)
│   └── __init__.py
│
├── utils/
│   ├── gcs_utils.py         # fetch_instructions(agent_name) from GCS
│   ├── context_utils.py     # fetch_document(file_name) from GCS context_store
│   └── focus_group_utils.py # write_json_data() to local output/ folder
│
├── Dockerfile               # python:3.11-slim, CMD: adk web --port $PORT
├── deploy.sh                # gcloud run deploy --source .
├── deploy-org.sh            # adk deploy cloud_run --with_ui
├── deploy-w-docker.sh       # gcloud run deploy with --set-env-vars
├── requirements.txt         # Full pinned dependency list
├── .env / .env_example      # GOOGLE_GENAI_USE_VERTEXAI + GOOGLE_API_KEY
├── Adk N8N Hybrid v4.json   # N8N workflow for ADK hybrid integration
└── CLAUDE TRAINING GUIDES/  # Mission context docs (read-only)
```

---

## Agent Inventory

| Agent | Type | Tools | Key Pattern |
|-------|------|-------|-------------|
| `greeting_agent` | `Agent` | None | Baseline; inline instruction |
| `calc_agent` | `Agent` | `BuiltInCodeExecutor` | Code execution; GCS callable instruction |
| `jarvis_agent` | `Agent` | `google_search` | Web search; GCS callable instruction |
| `travel_agent` | `SequentialAgent` | `google_search` (per sub-agent) | Sequential pipeline; `output_key` + template vars |
| `parallel_agent` | `ParallelAgent` | `google_search` (per sub-agent) | Concurrent execution; `output_key` |
| `combo_agent` | `Agent` | `AgentTool(SequentialAgent)` | Agent-as-tool nesting; chatbot front-end |
| `focus_group_agent` | `SequentialAgent` | `FunctionTool` (write_json) | 7-persona parallel; file output; `SequentialAgent` orchestration |
| `ghl_mcp_agent` | `Agent` | `MCPToolset` (HTTP) | MCP external service; LiteLlm; dynamic instruction |

---

## Agent Type Flow Diagrams

### Simple Agent
```
User Input
    ↓
Agent (LLM + tools)
    ↓
Response
```

### Sequential Agent
```
User Input
    ↓
Sub-Agent 1 → output_key="key1" → stored in session context
    ↓
Sub-Agent 2 uses {key1} in instruction template
    ↓
Response
```

### Parallel Agent
```
User Input
    ↓
┌── Sub-Agent 1 → output_key="key1"
├── Sub-Agent 2 → output_key="key2"   (all concurrent)
└── Sub-Agent N → output_key="keyN"
    ↓
All outputs in session context
    ↓
Response (last agent or orchestrator synthesizes)
```

### Combo Agent (combo_agent pattern)
```
User Input
    ↓
Root Agent (chatbot)
    ↓ (calls AgentTool when needed)
ResearchWorkflow (SequentialAgent)
    ├── ParallelAgent
    │   ├── NewsAgent → output_key="latest_news"
    │   └── StockAgent → output_key="current_stock_price"
    └── SynthesisAgent (uses {latest_news} + {current_stock_price})
    ↓
Response back to Root Agent
    ↓
Final Response to User
```

### Focus Group Agent
```
User Input (product/ad copy)
    ↓
SequentialAgent (DataCollector)
    ├── ParallelAgent (DockBloxxFocusGroup)
    │   ├── SamiDavis → output_key="sami_opinion"
    │   ├── MarkJohnson → output_key="mark_opinion"
    │   ├── LilaChen → output_key="lila_opinion"
    │   ├── GregEvans → output_key="greg_opinion"
    │   ├── ChloePatterson → output_key="chloe_opinion"
    │   ├── AlexRodriguez → output_key="alex_opinion"
    │   └── BrendaWhite → output_key="brenda_opinion"
    └── JsonWriterAgent (calls FunctionTool to write output to file)
    ↓
focus_group_results.json in output/focus_group/
```

---

## Configuration System

Minimal config. Two env vars drive everything:

```bash
GOOGLE_GENAI_USE_VERTEXAI=FALSE   # Use Gemini API directly (not Vertex)
GOOGLE_API_KEY=...                 # Gemini API key
```

For MCP/GHL agent, additional env vars:
```bash
GHL_API_TOKEN=...
GHL_LOCATION_ID=...
OPENROUTER_API_KEY=...
```

Loaded via `python-dotenv` from `.env` file.

---

## Deployment Architecture

### Path 1: Simple (Dockerfile-based)
```
Local code
    ↓
gcloud run deploy --source .   (uses Dockerfile automatically)
    ↓
Cloud Run Service
    CMD: adk web --port $PORT
```

### Path 2: ADK Native
```
Local code
    ↓
adk deploy cloud_run --with_ui --project --region --service_name
    ↓
Cloud Run Service (with ADK UI included)
```

### Cloud Run Execution
```
$PORT env var (provided by Cloud Run)
    ↓
adk web --port $PORT
    ↓
FastAPI server (ADK built-in)
    ↓
Serves agent via HTTP
```

---

## GCS Live Instructions Architecture

```
GCS Bucket: adk-agent-context-ninth-potion-455712-g9
└── ADK_Agent_Bundle_1/
    ├── calc_agent/
    │   └── calc_agent_instructions.txt
    ├── jarvis_agent/
    │   └── jarvis_agent_instructions.txt
    ├── greeting_agent/
    │   └── greeting_agent_instructions.txt
    └── context_store/
        └── moose_resume.txt  (arbitrary knowledge base docs)
```

**How it works:**
1. `utils/gcs_utils.py::fetch_instructions(agent_name)` — reads instruction `.txt`
2. Agent uses `instruction=callable_function` (not `instruction=called_result`)
3. ADK calls the function on every agent run → always fresh from GCS
4. Update instructions in GCS → no code deployment needed

---

## Module Entry Point Pattern

Every agent folder has `__init__.py`:
```python
from .agent import root_agent
```

ADK discovers `root_agent` via this import. The `adk run`, `adk web`, and `adk deploy` commands all look for `root_agent` in the agent's `__init__.py`.

---

_Extracted: 2026-02-23_
_Repo: google-adk-exp-v2_
