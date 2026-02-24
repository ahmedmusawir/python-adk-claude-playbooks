# CONSOLIDATION MISSION BRIEF

**To:** Claude Code (the instance doing the manual writing)
**From:** 10 prior Claude instances who extracted the patterns + User
**Date:** 2026-02-24
**Phase:** Phase 2 — Manual Creation (Extraction is DONE)

---

## Why You're Here

The user built Next.js apps in **2 hours** by giving Claude Code solid docs about their starter kit.
The user built VidGen (Python/AI) **without docs** and Claude fell on its ass.
**The variable was documentation.**

You are here to write Python/ADK documentation at the same quality level as those Next.js docs.
When you're done, another Claude Code instance will be unleashed into a Python project with your manuals — and it should build that app the same way: fast, confident, no guessing.

---

## What You Have (Your Inputs)

### 1. Repo Extractions (10 repos, fully analyzed)

| Repo | Type | Key Extractions |
|------|------|----------------|
| VidGen (project-bibo-youtube-v2) | Media pipeline | Vertex AI (Gemini/Imagen/TTS/STT), file-based state, sequential stages |
| crawl4ai-exp-project-v1 | RAG pipeline | crawl4ai, LangChain, Chroma, multi-provider LLM factory |
| google-adk-exp-v2 | ADK patterns lab | All ADK agent types, GCS callable instructions, output_key data flow |
| google-adk-n8n-hybrid-v2 | Production ADK + Cloud Run | N8N gateway, Supabase session, source-based deploy, .gcloudignore |
| google-adk-wrapper-v2 | FastAPI ADK wrapper | /run_agent endpoint, event parsing, session 404 auto-recovery |
| google-adk-n8n-hybrid-streamlit-v2 | Streamlit frontend | Supabase auth gate, gatekeeper(), per-user sessions, Mission Control |
| managed-rag-google-file-search-api-v1 | Google managed RAG | File Search API lifecycle, force=True delete, dual-document strategy |
| crawl4ai-exp-project-v1 (GHL chatbot) | RAG chatbot | Synthetic master index, system prompt as retrieval, split-panel layout |
| ghl-adk-agent-w-mcp-server-v1 | ADK + TypeScript MCP | MCPToolset, Streamable HTTP MCP, multi-runtime Docker, Nginx SSE |
| ghl-email-html-agent-v1 | Streamlit HTML editor | Global tool state, AgentTool, InMemorySession fix, BeautifulSoup tools |

Each repo has:
- `session-YYYY-MM-DD-{name}.md` — full analysis, patterns, gotchas, insights
- `/docs/architecture.md` — system flow
- `/docs/patterns.md` — copy-pasteable patterns
- `/docs/decisions.md` — key decisions with rationale
- `/docs/{integration}.md` — deep-dive on the major integration

### 2. The 35 Confirmed Cross-Repo Patterns

In `MISSION_BRIEF.md` under "Confirmed Cross-Repo Patterns" — these are the **battle-tested** patterns. High confidence for manuals. Use these as the backbone.

### 3. The Next.js Docs (FORMAT REFERENCE — Critical)

In the sample folder the user provided. Read these FIRST before writing a single manual. They are the **quality bar and format reference**. Notice:
- How they present copy-pasteable code snippets
- How they explain decisions (not just code, but *why*)
- How they handle "gotchas" and edge cases
- Their depth level (practical, not academic)
- How they're structured for a builder who needs to ship fast

**Your manuals must feel like those docs.** Not like academic papers. Not like "intro to X" tutorials. Like: "here's exactly what to do, here's the code, here's what will break you, go build."

---

## What You're Building (Your Outputs)

Create these manuals. Each one is a standalone reference a builder can read and immediately use.

### Manual Set: Python/ADK App Factory Docs

#### MANUAL 1: `adk-agents-fundamentals.md`
**Who uses it:** Anyone building their first ADK agent
**What it unlocks:** Build any single-agent app from scratch in < 30 min

Core content:
- Agent anatomy (`name`, `model`, `instruction`, `description`, `tools`, `output_key`)
- `Runner` + `InMemorySessionService` setup (global scope — the amnesia fix)
- `get_session()` None check pattern
- `asyncio.run()` sync wrapper
- `GOOGLE_GENAI_USE_VERTEXAI=1` + env vars
- Event loop: how to read events, extract text, detect tool calls
- `status_callback` pattern for tool execution feedback
- Agent module structure (`agent.py` + `__init__.py`, `root_agent` export)
- Running with `adk web` vs custom runner

#### MANUAL 2: `adk-advanced-patterns.md`
**Who uses it:** Builders who need multi-agent, specialized tools, or external integrations
**What it unlocks:** Sequential/Parallel agents, AgentTool, MCPToolset, LiteLlm

Core content:
- Sequential agent (pipeline of agents)
- Parallel agent (fan-out tasks)
- Combo agent (sequential + parallel mixed)
- AgentTool — Agent-as-a-Tool (Grounding + FunctionTool mix fix — critical ADK gotcha)
- MCPToolset + StreamableHTTPConnectionParams (connecting to MCP server)
- `tool_filter` for selective MCP tool exposure
- `output_key` for passing data between agents
- LiteLlm agent (multi-provider alternative)
- GCS callable instructions (`instruction=fn` not `instruction=fn()`)
- Sequential tool execution instruction (prevents parallel MCP call session poisoning)

#### MANUAL 3: `vertex-ai-integrations.md`
**Who uses it:** Anyone using Vertex AI capabilities beyond basic text
**What it unlocks:** All AI modalities in one place

Core content:
- Gemini text generation (standard + streaming)
- Imagen image generation (model names, parameters, output handling)
- Text-to-Speech (Vertex AI TTS, voice list, SSML)
- Speech-to-Text (Vertex AI STT, audio format requirements)
- `GOOGLE_GENAI_USE_VERTEXAI=1` vs explicit `Client(vertexai=True)`
- `google.genai` vs `vertexai` SDK differences
- Auth: ADC vs explicit credentials
- Multi-modal inputs (`types.Part.from_bytes` for PDF/image analysis)
- Which models for which tasks (Gemini Flash vs Pro, Imagen 3, etc.)

#### MANUAL 4: `cloud-run-deployment.md`
**Who uses it:** Anyone deploying to Google Cloud
**What it unlocks:** From local to Cloud Run in one session

Core content:
- Source-based deploy (`gcloud builds submit` + `gcloud run deploy`)
- Secret Manager secrets (`--set-secrets=VAR=secret-name:latest`)
- `.gcloudignore` — what to exclude (node_modules, .venv, __pycache__)
- Service account + IAM setup (Vertex AI permissions)
- `--allow-unauthenticated` vs invoker SA (+ when to use each)
- `--no-cpu-throttling --cpu-boost` for streaming/AI workloads
- `--min-instances=1` for cold start prevention
- Shell form CMD in Dockerfile (env var substitution `$PORT`)
- Dockerfile for Python + optional Node.js
- `deploy.sh` template (complete working script)
- Cloud Run health checks

#### MANUAL 5: `fastapi-adk-gateway.md`
**Who uses it:** Anyone building an API layer over an ADK agent
**What it unlocks:** Expose any agent as a clean REST API

Core content:
- `/run_agent` endpoint (the canonical pattern)
- Request/response contract (user_input, session_id, return effective_session_id)
- ADK session creation + `/run` call + event parsing — all in one endpoint
- Session 404 auto-recovery (retry with new session on error)
- History normalization (convert ADK events to [{role, content}] format)
- `config.json` env-aware agent registry
- Two-service architecture (FastAPI wrapper + ADK web)
- CORS setup for Streamlit/Next.js clients

#### MANUAL 6: `streamlit-patterns.md`
**Who uses it:** Anyone building a Streamlit UI for an AI agent
**What it unlocks:** Complete UI pattern library for agent frontends

Core content:
- Two-screen app (load/setup → main editor)
- Sidebar chat + main preview layout
- `st.container(height=N)` for scrollable chat history
- `st.chat_input` + `st.chat_message` pattern
- `st.toast()` for non-blocking tool feedback
- Undo stack (`st.session_state.history` as list)
- Live HTML preview (`components.html()`)
- Supabase auth gate (`session is None` → login form → `st.rerun()`)
- `gatekeeper()` for multi-page access control
- Per-user session bookmarks (Supabase upsert of JSON blob)
- Mission Control admin panel (live GCS prompt editing from browser)
- `Procfile` for Cloud Run deployment
- `st.secrets` → Secret Manager dual path (no code change between envs)
- Agent session ID in `st.session_state` (amnesia prevention)
- `asyncio.run()` wrapper for async ADK calls in Streamlit

#### MANUAL 7: `rag-pipelines.md`
**Who uses it:** Anyone building a document Q&A or knowledge base feature
**What it unlocks:** Both DIY and managed RAG approaches

Core content:
- **DIY RAG (crawl4ai + LangChain + Chroma):**
  - crawl4ai scraping API (AsyncWebCrawler, CrawlerRunConfig)
  - Text chunking + embedding + Chroma vector store
  - Retrieval + LLM synthesis
  - Multi-provider LLM factory (LangChain abstraction)
  - site_config naming convention
- **Managed RAG (Google File Search API):**
  - Create file store → async upload → poll → query → delete lifecycle
  - `force=True` delete gotcha
  - Dual-document strategy (original + structured summary)
  - Synthetic master index for aggregation queries
  - System prompt as retrieval strategy
  - Filename-based category extraction (over LLM parsing)
  - Batch upload with per-file error handling
- **Trade-off guide:** When to use DIY vs managed

#### MANUAL 8: `mcp-server-typescript.md`
**Who uses it:** Anyone building a TypeScript MCP server for ADK to consume
**What it unlocks:** Expose any API as MCP tools for any ADK agent

Core content:
- Old SSE vs new Streamable HTTP (migration guide)
- `McpServer` + `registerTool()` pattern (complete setup)
- Zod schema format (required — JSON schema not accepted)
- `sessionIdGenerator: undefined` for stateless mode
- Error-as-response: `{ isError: true }` — never throw (session poisoning gotcha)
- 30s timeout wrapper pattern
- Tool category class pattern (organizing 20+ tools)
- `locationId` injection at API client (never surface to agent)
- Health check + `/capabilities` endpoints
- `enableJsonResponse: true` flag

#### MANUAL 9: `multi-runtime-docker.md`
**Who uses it:** Anyone combining Python AI + Node.js services in one container
**What it unlocks:** Single Cloud Run container with multiple runtimes

Core content:
- Python 3.12-slim + `apt-get install nodejs` base image
- Supervisor for multi-process orchestration (`nodaemon=true` — required for Docker)
- Nginx as sole public port (reverse proxy to internal services)
- Nginx SSE streaming config (`proxy_buffering off`, timeout, HTTP 1.1)
- Nginx Basic Auth setup (`htpasswd` in Dockerfile)
- Complete `supervisord.conf` template
- Complete `nginx.conf` template
- `build npm` inside Dockerfile (TypeScript compilation at build time)
- Port assignments: Nginx=8080 (public), ADK=8000, MCP=9000
- `.gcloudignore` for node_modules

#### MANUAL 10: `supabase-integration.md`
**Who uses it:** Anyone adding auth or persistence to a Python/Streamlit app
**What it unlocks:** Auth gate, user profiles, session persistence

Core content:
- Email/password auth gate in Streamlit
- `supabase.auth.sign_in_with_password()` + `st.rerun()` pattern
- `gatekeeper()` function for multi-page protection
- Per-user session bookmarks (upsert JSON blob)
- ADK session persistence (store session_id per agent per user)
- Supabase in Cloud Run (secrets injection)
- `st.secrets` → Secret Manager dual path for Supabase keys

---

## How to Write These Manuals

### The Format That Works (Based on Next.js Docs)

Study the sample Next.js docs before writing. Then match this structure for each manual:

```
# Manual Title

## What This Is For
One paragraph. What problem does this manual solve? Who reads it?

## Quick Reference
A table or code block showing the most common 80% use case.
Someone should be able to copy this and get running.

## Core Concepts
Brief explanation of the mental model. Not academic — builder-focused.
"Think of it as X" style explanations.

## Patterns
Each pattern:
- Name
- When to use it
- Complete code (copy-pasteable, not pseudocode)
- What to watch out for (gotchas)

## Gotchas (Critical)
The things that WILL break you if you don't know them.
These come from real failures across the 10 repos.
Be specific. Include the error message if known.

## Complete Working Example
A minimal but complete working implementation.
Not a toy — something that actually runs.

## What's Next / Related Manuals
Point to the next manual the reader likely needs.
```

### Quality Rules

1. **Copy-pasteable code only** — no pseudocode, no `...`, no `# your code here`. If you can't show the complete pattern, explain why and show the closest complete example.

2. **Every gotcha must be real** — from actual failures documented in the 10 session files. No hypothetical warnings.

3. **No theory for its own sake** — explain *why* only when it changes what the builder does. "Session service must be global because X" is useful. "Here's how ADK session management works architecturally" is not.

4. **Match the consumer's mental model** — the consumer is a Claude Code instance that will receive an app brief and needs to build it. Write for that reader. What does it need to know to execute, not to understand?

5. **Patterns over prose** — a page of bullet points with code > three paragraphs of explanation.

6. **Confirmed patterns only** — if something only appeared in one repo and wasn't tested in production, flag it. If it appeared in 3+ repos, present it as the standard.

---

## What NOT to Do

- Don't write "intro to AI" content — assume the reader knows Python and has used an LLM API before
- Don't include patterns you haven't seen in the repos — stick to what was actually built
- Don't write theoretical alternatives — present the pattern that worked, note alternatives only if you saw both fail and succeed
- Don't over-structure — if a manual is getting too long, split it; if too short, merge it
- Don't copy-paste from session files directly — synthesize. The session files are raw notes; the manuals are the polished output.

---

## Priority Order

Write manuals in this order (highest impact first):

1. `adk-agents-fundamentals.md` — needed for every ADK app
2. `streamlit-patterns.md` — most common UI layer
3. `cloud-run-deployment.md` — needed for every production deploy
4. `adk-advanced-patterns.md` — needed for complex agents
5. `vertex-ai-integrations.md` — needed for multi-modal apps
6. `fastapi-adk-gateway.md` — needed for API-exposed agents
7. `rag-pipelines.md` — needed for knowledge base apps
8. `mcp-server-typescript.md` — needed for tool-heavy agents
9. `multi-runtime-docker.md` — needed for Python+Node.js deploys
10. `supabase-integration.md` — needed for auth/persistence

---

## The Definition of Done

A manual is **done** when:
- Another Claude Code instance can read it and build the thing without looking at any repo
- Every code snippet is runnable (not pseudocode)
- Every gotcha that caused a real failure is documented
- The format matches the Next.js docs quality bar

The whole manual set is **done** when:
- A new app brief comes in
- Claude Code reads the relevant manuals
- Claude Code builds the backend as fast as it built the Next.js frontend
- User doesn't fall from their chair in frustration

---

## Context on the Broader System

```
Architect Agent (chatbot mode)
    ↓
app_brief.md + ui_specs.md
    ↓
Designer Agent → Stitch → designs
    ↓
Claude Code + Next.js docs → frontend (already works, 2 hours)
    ↓
Claude Code + YOUR MANUALS → backend (this is what you're enabling)
```

You are writing the docs that make the backend half of this factory work.
The frontend already works perfectly. The backend is the bottleneck.
Fix the bottleneck.

---

## Folder Structure to Create

All manuals go in the `/docs/` folder of whatever repo you're working in:

```
/docs/
├── adk-agents-fundamentals.md
├── adk-advanced-patterns.md
├── vertex-ai-integrations.md
├── cloud-run-deployment.md
├── fastapi-adk-gateway.md
├── streamlit-patterns.md
├── rag-pipelines.md
├── mcp-server-typescript.md
├── multi-runtime-docker.md
└── supabase-integration.md
```

---

_Written by: Claude Code Sonnet 4.6 (2026-02-24)_
_Based on: 10 repo extractions, 35 confirmed patterns, Next.js docs format reference_
_Purpose: Enable fast Python/ADK backend builds in the AI App Factory_
