# Key Decisions: google-adk-exp-v2

---

## Decision 1: One Agent Per Folder

**Context:** ADK requires a discoverable `root_agent`. Could put all agents in one file or split into folders.

**Decision:** Each agent gets its own folder (`agent_name/agent.py` + `agent_name/__init__.py`).

**Rationale:**
- ADK `run`/`web`/`deploy` commands target a folder, not a file
- Isolation: each agent is independently runnable
- Clean import path: `from greeting_agent import root_agent`
- Easy to add agent-specific utils without polluting global space

**Alternative rejected:** All agents in one flat file — breaks ADK discovery pattern.

---

## Decision 2: `output_key` for Inter-Agent Communication (Not Direct Calls)

**Context:** Sequential agents need to pass data between stages. Options: direct function calls, return values, or session context.

**Decision:** Use `output_key="key_name"` on each sub-agent to write to session context, then reference `{key_name}` in downstream instruction templates.

**Rationale:**
- Native ADK mechanism — how the framework was designed
- Decoupled: agents don't know about each other's internals
- Observable: session context is inspectable for debugging
- Template substitution is automatic — ADK handles it

**Alternative rejected:** Explicit function calls between agents — fights the framework, loses ADK's session management.

---

## Decision 3: Callable Instruction (Not Static String) for Dynamic Prompts

**Context:** Agent instructions may need to change without code deployment (e.g., tune the prompt for a client).

**Decision:** Use `instruction=function_ref` (callable) rather than `instruction="static string"` or `instruction=function_call()`.

**Rationale:**
- Called on every agent run → always fresh content
- Enables prompt updates without redeployment (massive ops win)
- GCS as instruction store = version-controlled prompts outside codebase
- The difference `my_fn` vs `my_fn()` is subtle but critical

**Alternative rejected:** `instruction=fetch_instructions("agent")` — evaluated once at startup, cached forever, requires server bounce to update.

**Anti-pattern (do not use):**
```python
instruction=fetch_instructions("agent")  # ← CACHED AT STARTUP
```
**Correct pattern:**
```python
def get_instructions(ctx): return fetch_instructions("agent")
instruction=get_instructions  # ← FRESH EVERY RUN
```

---

## Decision 4: GCS for Live Instructions

**Context:** Where to store agent instructions that can be updated without code deployment?

**Decision:** Google Cloud Storage. File per agent: `BUCKET/FOLDER/agent_name/agent_name_instructions.txt`.

**Rationale:**
- No database required (consistent with file-based philosophy)
- GCS is already in the Google Cloud stack (no new dependency)
- Simple text files — editable by non-engineers
- ADC (Application Default Credentials) handles auth automatically in Cloud Run
- `context_store/` subfolder enables knowledge base documents as tools

**Alternatives rejected:**
- Firestore/BigQuery: overkill for a text file
- Hardcoded in code: requires redeployment on every prompt change
- Environment variables: size limits, not human-friendly to edit

**Trade-off:** Adds GCS latency to every agent run. Acceptable for most use cases.

---

## Decision 5: `AgentTool` for Nesting Workflows in Chatbots

**Context:** Want a conversational chatbot that delegates to complex multi-agent pipelines only when needed.

**Decision:** Wrap the workflow in `AgentTool(agent=workflow_agent)` and pass it to the chatbot's `tools=[]`.

**Rationale:**
- Chatbot handles small talk natively (no pipeline overhead)
- Pipeline only runs when the LLM decides it's relevant
- Clean separation: chatbot = conversation layer, workflow = execution layer
- ADK's `AgentTool` is the idiomatic way to compose agents hierarchically

**Alternative rejected:** Making the `SequentialAgent` the root agent directly — every message triggers the full pipeline, even "hello" messages.

---

## Decision 6: `ParallelAgent` for Independent Research Tasks

**Context:** Multiple independent data-gathering tasks (news + stock, or 7 focus group personas).

**Decision:** Use `ParallelAgent` with one sub-agent per independent task.

**Rationale:**
- True concurrency — all branches run simultaneously
- Massive time saving when each branch takes 2-5 seconds
- Each agent has its own `output_key` so results don't collide
- Follows ADK's design intent for parallel workloads

**Alternative rejected:** Sequential execution of independent tasks — unnecessary serialization, 7x slower for 7-persona focus group.

---

## Decision 7: Gemini 2.5 Flash as Default Model

**Context:** Multiple model options (1.5-flash, 2.0-flash, 2.5-flash, Pro versions).

**Decision:** Default to `gemini-2.5-flash`. Use `gemini-2.0-flash` for simpler sub-agents.

**Rationale:**
- 2.5-flash: Best current quality/speed/cost balance for most tasks
- 2.0-flash: Sufficient for simple retrieval/formatting sub-agents, lower cost
- Flash > Pro for latency-sensitive production use

**Model selection guide observed in codebase:**
- `gemini-2.5-flash` — orchestrator agents, synthesis, complex reasoning
- `gemini-2.0-flash` — retrieval agents, persona agents, simple tasks
- `gemini-1.5-flash` — legacy, present in travel_agent (old code, should upgrade)

---

## Decision 8: LiteLlm for Non-Gemini Models

**Context:** GHL CRM agent wanted GPT-5 / Claude via OpenRouter. ADK natively uses Gemini.

**Decision:** Use `google.adk.models.lite_llm.LiteLlm` wrapper with OpenRouter.

**Rationale:**
- Native ADK integration — `model=lite_llm_client` just works
- LiteLlm is a unified interface for 100+ LLM providers
- OpenRouter as the router = one API key, many models
- No custom API client needed — ADK handles the rest

**Alternative rejected:** Separate non-ADK code path for non-Gemini models — complexity, duplicated session management.

**Note:** Model ID in code has a typo: `"openrcouter/..."` — should be `"openrouter/..."`. This is a live bug in ghl_mcp_agent/agent.py.

---

## Decision 9: MCPToolset for External Service Integration

**Context:** Integrate GoHighLevel CRM into an agent. Options: custom REST client, LangChain tool, MCP.

**Decision:** Use `MCPToolset` with `StreamableHTTPConnectionParams`.

**Rationale:**
- GHL exposes an MCP server — use it natively
- MCP auto-discovers all available tools (no manual tool registration)
- `tool_filter` provides safety valve to limit exposed tools
- Credentials passed via headers (standard HTTP auth)
- Future-proof: any service with MCP support = one pattern for all

**Alternative rejected:** Custom FunctionTool wrapping REST calls — requires manual implementation and maintenance of every API endpoint.

---

## Decision 10: `FunctionTool` for File Output in Agent Pipelines

**Context:** Agent pipeline (focus group) needs to write results to a file. Options: agent writes file directly, or tool handles it.

**Decision:** Wrap file-writing logic in `FunctionTool` and let the LLM agent call it.

**Rationale:**
- ADK agents execute tools, not raw Python — respects the framework
- LLM agent can format the data before writing (JSON serialization)
- Graceful fallback: if LLM returns malformed JSON, wrap in `{"raw_output": ...}`
- Tool docstring tells the LLM exactly when and how to use it

---

## Decision 11: Two Deployment Paths (gcloud vs adk deploy)

**Context:** Need to deploy agents to Cloud Run. ADK has its own deploy command, but `gcloud` is more universal.

**Decision:** Maintain both paths:
- `deploy.sh` → `gcloud run deploy --source .` (uses Dockerfile, simple)
- `deploy-org.sh` → `adk deploy cloud_run --with_ui` (ADK native, includes UI)

**Rationale:**
- `gcloud run deploy --source .` works with the Dockerfile directly, full control
- `adk deploy cloud_run` is ADK-idiomatic, auto-includes the ADK web UI
- Different use cases: API-only deployment vs. UI-included deployment

**Rule of thumb:**
- Use `adk deploy cloud_run --with_ui` for demos / internal tools (UI included)
- Use `gcloud run deploy` for production APIs (Dockerfile control, no UI)

---

## Decision 12: `python:3.11-slim` Base Image

**Context:** Dockerfile base image choice.

**Decision:** `python:3.11-slim` — Python 3.11, minimal image.

**Rationale:**
- ADK (google-adk==1.13.0) requires Python 3.11+
- `slim` variant: smaller image, faster deploys, less attack surface
- `.python-version` file in repo pins to Python 3.11 for local dev consistency

---

_Extracted: 2026-02-23_
_Repo: google-adk-exp-v2_
