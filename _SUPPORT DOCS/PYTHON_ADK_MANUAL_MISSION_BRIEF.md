# MISSION BRIEF — Python/ADK Manual Creation

**To:** Claude Code (the instance reading this right now)
**From:** The user + 10 prior Claude extraction instances
**Read time:** ~5 minutes
**After reading this:** You will know exactly what to do. No guessing required.

---

## The Short Version

You are going to write a set of Python/ADK reference manuals — the same way someone wrote Next.js docs for this user's starter kit, which enabled Claude to build full production apps in 2 hours.

The Next.js docs are in this folder. Read them. That is your quality bar.
The Python pattern extractions are in this folder. That is your source material.
The target app is described below. That is your north star.

When you are done, another Claude Code instance should be able to read your manuals and build a full Python/ADK backend as fast as the frontend gets built today.

---

## Why This Exists (Read This — It Matters)

The user built **VidGen** (a Python/AI app) without proper docs. Claude was unleashed into that project with no harness. It failed. Not because Claude is dumb — because it had no reference point, no patterns, no battle-tested code to lean on. It had to guess at everything.

The user then did something remarkable: **they documented the failure**. They did a brain dump *on Claude itself* — asking it to explain why it failed, what it needed, how a proper harness should be built. Those docs are in this folder under the `vidgen-failure-docs/` folder (or similar). **Read those first after this brief.** They are your own words describing your own limitations. They are the most honest description of what makes you fail and what makes you succeed.

Meanwhile, the frontend side of this factory works perfectly. The user has Next.js docs for a production starter kit (ecom, auth, Supabase, GHL CRM, etc.). When Claude Code gets those docs + an app brief + designer output, it builds production-quality apps in about 2 hours. The user literally fell out of their chair watching it work.

**The variable is documentation.**

Your job: close the gap. Make the Python/ADK backend side work the same way.

---

## What's In This Folder (Your Complete Inventory)

Read everything. Not skimming — actually read it. Your manuals will only be as good as your understanding of what's here.

### 📁 Reference: Format & Quality Bar
- **`nextjs-docs/`** — The Next.js starter kit docs. This is your format reference. Study the structure, depth, code style, and how gotchas are called out. Your manuals must feel like these docs.
- **`nextjs-ecom-app-docs/`** — Docs for a production ecom app (making money right now). Shows you what a real production-grade doc set looks like for a complete app.

### 📁 Source: What Went Wrong
- **`vidgen-failure-docs/`** — Post-mortem brain dump from a prior Claude instance about why it failed building VidGen. Your own words. Describes what the harness needs to be, how a good agent environment should be structured. Pure gold. Read this before writing anything.
- **`vidgen-app-brief.md`** (or similar) — The original app brief used to build VidGen. Cascade eventually built it successfully. Shows you what a real app brief looks like and what it demands from the backend.

### 📁 Source: Repo Extractions (10 Repos)
These are the pattern extraction outputs from 10 production Python/AI repos. Each has session notes, architecture docs, pattern libraries, and decision logs.

The 3 production apps the user cares about most (reference these constantly):
1. **GHL Agent + MCP Server** (`session-2026-02-24-ghl-adk-mcp-server.md` + its `/docs/`) — Production CRM agent in Cloud Run. ADK + TypeScript MCP + Docker multi-runtime. The most complete production deployment pattern.
2. **Managed RAG GHL API Docs Chatbot** (`session-2026-02-23-managed-rag-google-file-search.md` + `session-2026-02-23-ghl-rag-chatbot.md`) — Production Streamlit RAG chatbot. Shows the full Google managed RAG lifecycle plus a real Streamlit UI.
3. **ADK Agent Bundle** (`session-2026-02-23-google-adk-n8n-hybrid.md` + `session-2026-02-23-google-adk-wrapper-v2.md` + `session-2026-02-23-google-adk-n8n-hybrid-streamlit-v2.md`) — This was extracted as 3 repos but it's one production system: ADK backend + FastAPI ADK wrapper middleware + Streamlit frontend. Production in Cloud Run. **This is the closest to the user's dream architecture.**

The other 7 repos fill in pattern gaps (see the extraction session files for each).

### 📁 Source: The 35 Battle-Tested Patterns
In `MISSION_BRIEF.md` (the extraction phase brief), section "Confirmed Cross-Repo Patterns." These appeared in multiple repos. High confidence. Use as backbone of every manual.

### 📁 Reference: Design Output
- **`vidgen-designs/`** — Output from the designer agent + Stitch for VidGen. Shows what the design side of the factory produces. Calibrates what Claude Code needs to build against.

### 📁 Reference: Failed Build
- The crawl4ai RAG build (LangChain + Chroma) was abandoned when the user switched to Google managed RAG. The session file for it (`session-2026-02-23-crawl4ai-rag.md`) documents why. Useful for the RAG manual's "DIY vs managed" trade-off section.

---

## What You Are Building

### 10 Python/ADK Reference Manuals

All go in `/docs/` of whatever repo you're working in.

---

### MANUAL 1: `adk-agents-fundamentals.md` ← START HERE
**Unlocks:** Any single-agent ADK app from scratch in < 30 min

Must cover:
- Agent anatomy (all constructor fields with what each does)
- `Runner` + `InMemorySessionService` at **module scope** (the amnesia fix — critical)
- `get_session()` returns `None`, not exception — explicit check required
- `asyncio.run()` sync wrapper pattern for Streamlit/FastAPI compatibility
- `GOOGLE_GENAI_USE_VERTEXAI=1` env var (set at top of `agent.py` before imports)
- `GOOGLE_CLOUD_PROJECT` + `GOOGLE_CLOUD_LOCATION` required
- Event loop reading: text extraction + `function_call` detection
- `status_callback` pattern for tool execution feedback in UI
- Agent module structure: `agent.py` + `__init__.py` + `root_agent` export
- `adk web` vs `adk api_server` (the user prefers `adk api_server` — FastAPI mode)
- Complete working minimal agent (copy-paste and run)

---

### MANUAL 2: `adk-advanced-patterns.md`
**Unlocks:** Multi-agent, specialized tools, external integrations

Must cover:
- Sequential, Parallel, Combo agent patterns with code
- **AgentTool (Agent-as-a-Tool)** — the Grounding + FunctionTool mixing fix (ADK throws `400 INVALID_ARGUMENT` otherwise — critical gotcha)
- MCPToolset + StreamableHTTPConnectionParams (external MCP server connection)
- `tool_filter` for selective MCP tool exposure
- `output_key` for inter-agent data flow
- LiteLlm agent (for multi-provider or non-Vertex models)
- GCS callable instructions: `instruction=fn` NOT `instruction=fn()` (GCS-fetched system prompts)
- Named agent protocols in system prompt (Protocol Phoenix, Protocol 10 patterns)
- Sequential tool execution instruction (prevents parallel MCP call session poisoning)
- Global in-module tool state pattern (`_active_html` bridge — for document editing agents)

---

### MANUAL 3: `vertex-ai-integrations.md`
**Unlocks:** All AI modalities (text, image, audio, vision, search)

Must cover:
- Gemini text generation (standard + streaming)
- Imagen image generation (model IDs, parameters, output formats)
- Vertex AI TTS: voice list, SSML, audio format
- Vertex AI STT: audio format requirements, async handling
- `types.Part.from_bytes` for PDF and image direct analysis (multimodal)
- `google.genai` SDK vs `vertexai` SDK — when to use which
- ADC (Application Default Credentials) vs explicit credentials
- Vertex AI auth for Cloud Run (service account + roles)
- Model selection guide: Gemini Flash vs Pro, which for which task
- **Vertex Context Caching** — cache expensive context (system prompts, large docs) across calls; reduces latency + cost on repeated queries with same context
- **Google Files API (Live Context Upload)** — upload files at runtime for the model to reference; useful for per-session document analysis; different from File Search API (which is RAG)

---

### MANUAL 4: `cloud-run-deployment.md`
**Unlocks:** Local to Cloud Run in one session, every time

Must cover:
- Complete `deploy.sh` template (copy-paste and run)
- Source-based deploy: `gcloud builds submit` + `gcloud run deploy`
- `.gcloudignore` essentials (node_modules, .venv, __pycache__, .git)
- Secret Manager: `--set-secrets=VAR=secret-name:latest` syntax
- Service account setup + Vertex AI permissions
- `--no-cpu-throttling --cpu-boost` for AI/streaming workloads
- `--min-instances=1` to prevent cold start race conditions
- Shell form CMD in Dockerfile: `CMD adk api_server --port $PORT` (env var substitution)
- `--allow-unauthenticated` vs invoker service account (when to use each)
- Health check endpoint requirements for Cloud Run
- Multi-stage builds (if needed)

---

### MANUAL 5: `fastapi-adk-gateway.md` ← User's Dream Backend
**Unlocks:** Every ADK agent as a clean REST API, consumable from Next.js

This is the user's preferred architecture. All agents eventually served as APIs.

Must cover:
- `adk api_server` command — what it starts, what it exposes, default port
- The `/run_agent` wrapper pattern (custom FastAPI over ADK)
- Request/response contract (user_input, session_id → response_text, effective_session_id)
- Session creation + `/run` + event parsing — all in one endpoint
- Session 404 auto-recovery (detect error → create new session → retry)
- History normalization: ADK events → `[{role, content}]` format for frontend
- `config.json` env-aware agent registry pattern
- CORS setup for Next.js / Streamlit clients
- Two-service architecture: FastAPI wrapper on 8080, ADK web on 8000 (internal)
- How Next.js consumes this API (fetch patterns, session ID management)
- Auth: adding API key middleware to FastAPI (cloud-ready pattern)

---

### MANUAL 6: `streamlit-patterns.md`
**Unlocks:** Complete Streamlit UI for any AI agent app

Must cover:
- Two-screen pattern: load/setup → main app (single file, zero routing)
- Sidebar chat + main preview/output layout
- `st.container(height=N)` for scrollable chat history
- `st.chat_input` + `st.chat_message` loop pattern
- `st.toast()` for non-blocking tool execution feedback
- Undo stack (`st.session_state.history` list + `save_state()` + `undo()`)
- Live HTML/content preview (`components.html()`)
- Supabase auth gate: `if session is None` → login form → `supabase.auth.sign_in_with_password()` → `st.rerun()`
- `gatekeeper()` for multi-page protection (one call = `st.stop()`)
- Per-user session bookmarks (upsert `{agent_name: session_id}` JSON blob)
- Mission Control admin panel (live GCS prompt editing from browser UI)
- Agent session ID in `st.session_state` + global `InMemorySessionService` (amnesia prevention)
- `Procfile` for Cloud Run: `web: streamlit run app.py --server.port $PORT`
- `st.secrets` → Secret Manager dual path
- `asyncio.run()` inside Streamlit (why it works, how to do it safely)

---

### MANUAL 7: `rag-pipelines.md`
**Unlocks:** Document Q&A, knowledge base features, chatbot with docs

Must cover both tracks:

**Track A — Google Managed RAG (Recommended):**
- File Search API full lifecycle: create store → async upload → poll done → query → delete
- Polling pattern: `while not operation.done: sleep(1); operation = client.operations.get(...)`
- `force=True` on delete (required when store has sub-resources — gotcha)
- Dual-document strategy: upload original + structured summary per document
- Synthetic master index for aggregation queries (category counts, overviews)
- System prompt as retrieval strategy (instruct model to use retrieval)
- Filename-based category extraction (more reliable than LLM parsing)
- API key vs Vertex AI auth for File Search API
- Batch upload with per-file error handling

**Track B — DIY RAG (crawl4ai + LangChain + Chroma):**
- crawl4ai: `AsyncWebCrawler`, `CrawlerRunConfig`, async scraping loop
- Text chunking + embedding + Chroma vector store setup
- Retrieval + LLM synthesis chain
- Multi-provider LLM factory pattern
- `site_config` naming convention

**Trade-off guide:** Managed = simpler, less control, Google-managed retrieval. DIY = full control, more ops work. Use managed unless you need custom chunking/retrieval.

---

### MANUAL 8: `mcp-server-typescript.md`
**Unlocks:** Expose any external API as MCP tools for any ADK agent

Must cover:
- Old SSE API vs new Streamable HTTP API (migration diff — the old way breaks)
- `McpServer` + `registerTool()` complete setup
- Zod schema format (required — raw JSON schema not accepted in new API)
- `sessionIdGenerator: undefined` for stateless mode (omitting causes tracking bugs)
- **Error-as-response: `{ isError: true }`** — NEVER throw in tool handlers (session poisoning — critical gotcha)
- 30s timeout wrapper with `Promise.race`
- Tool category class pattern (for 20+ tools: one class per domain)
- `locationId` (or any tenant ID) injection at API client layer (never surface to agent)
- Health check + `/capabilities` endpoints
- `enableJsonResponse: true` flag purpose
- Stdio vs HTTP transport (when to use which)
- How ADK's MCPToolset connects to this server

---

### MANUAL 9: `multi-runtime-docker.md`
**Unlocks:** Python AI + Node.js services in one Cloud Run container

Must cover:
- Python 3.12-slim + `apt-get install nodejs npm` base
- Supervisor: `nodaemon=true` (required for Docker — without it container exits immediately)
- Nginx as sole public port (reverse proxy to internal services)
- Nginx SSE streaming: `proxy_buffering off; proxy_cache off; proxy_read_timeout 300s;` (critical for ADK streaming)
- Nginx Basic Auth: `htpasswd -bc` in Dockerfile
- Port layout: Nginx=8080 (public), ADK/FastAPI=8000, MCP=9000
- Complete `supervisord.conf` template
- Complete `nginx.conf` template with SSE + auth
- TypeScript compilation at Docker build time (`npm install && npm run build`)
- Complete working Dockerfile template

---

### MANUAL 10: `supabase-integration.md`
**Unlocks:** Auth + persistence for any Python/Streamlit/FastAPI app

Must cover:
- Email/password auth gate in Streamlit (full pattern)
- `supabase.auth.sign_in_with_password()` + error handling + `st.rerun()`
- `gatekeeper()` multi-page protection
- Per-user session bookmarks: upsert `{agent_name: session_id}` JSON blob per user
- ADK session ID persistence across logins
- Supabase client setup with `st.secrets` + Secret Manager dual path
- Row-level security basics (user can only see own data)
- Supabase in Cloud Run: secrets injection pattern

---

## The Target App (Your North Star)

Everything you write should be calibrated toward enabling this app to be built fast:

### Agentic Harness — ADK Backend + Next.js Frontend

A full-featured AI agent platform. Every capability the user needs, in one app.

**Backend (ADK + FastAPI):**
- `adk api_server` as the deployment base
- FastAPI wrapper exposing a clean `/run_agent` endpoint
- Multiple agents registered in `config.json`
- All agents consumable from Next.js via API

**AI Capabilities:**
- **Vertex Context Caching** — cache system prompts or large reference docs across calls; huge latency + cost win for repetitive queries
- **Live Context Upload (Google Files API)** — user uploads a file at runtime; it's passed directly to the model for that session
- **Managed RAG (File Search API)** — persistent knowledge base; agent can query docs stored in a File Search store
- **Token Calculator** — track input/output token usage per call; display to user; enable cost awareness

**Memory & Sessions:**
- **7-day session file memory** — each session's key facts written to a file; loaded as context on next session; expires after 7 days; gives agent "memory" without a database
- **InMemorySessionService** for within-session conversation history (standard ADK pattern)
- Supabase for cross-device session bookmark persistence (user returns to same conversation on any device)

**Tooling:**
- FunctionTools (Python functions) for custom business logic
- MCPToolset for external tool servers (GHL CRM, etc.)
- AgentTool for specialist sub-agents (search, image generation, etc.)

**Skills:**
- On-demand skill loading: agent can load specialized instruction sets for specific task types
- Skills are stored as files (GCS or local) and injected into context when triggered
- Example: "legal review skill", "email writing skill", "code review skill"
- Triggered by user intent or explicit slash commands

**Frontend (Next.js):**
- Calls ADK FastAPI backend via REST
- Supabase auth
- Chat UI (streaming responses preferred)
- File upload for live context
- Session management (remember conversations)

**Why this is the north star:** Every manual you write should make this app faster to build. If you read your manuals and can build this app cleanly, the manuals are done.

---

## How to Work

### Priority Order
1. `adk-agents-fundamentals.md` — needed for every single ADK app
2. `streamlit-patterns.md` — most common UI layer in the series
3. `cloud-run-deployment.md` — every production app needs this
4. `fastapi-adk-gateway.md` — the user's dream architecture
5. `adk-advanced-patterns.md` — needed for complex agents
6. `vertex-ai-integrations.md` — multi-modal + context caching
7. `rag-pipelines.md` — knowledge base apps
8. `mcp-server-typescript.md` — tool-heavy agents
9. `multi-runtime-docker.md` — Python+Node deployments
10. `supabase-integration.md` — auth + persistence

### Format Rules (Non-Negotiable)

1. **Match the Next.js docs quality** — read them before writing anything. They are the bar.
2. **Copy-pasteable code only** — no pseudocode, no `...`, no `# your implementation here`. Complete patterns only.
3. **Every gotcha is real** — sourced from actual failures in the 10 session files. No hypothetical warnings.
4. **The VidGen brain dump is your gotcha source** — your own post-mortem explains exactly where and why things break. Mine it.
5. **Patterns over prose** — code block + 2 sentence explanation beats 3 paragraphs.
6. **Confirmed patterns get confidence labels** — if something appeared in 3+ repos, say "standard pattern." If only one repo, say "first seen in X."
7. **No intro-to-AI content** — assume reader has used LLM APIs before. Start from "here's the ADK-specific way."

### What NOT to Do
- Don't copy-paste session file content directly — synthesize it into manual format
- Don't include patterns you haven't seen in the repos
- Don't write manuals that read like documentation — write them like recipes
- Don't pad. If a manual is 2 pages, great. If 10 pages, also great. No filler.

---

## Definition of Done

**A manual is done when:** Another Claude Code instance can read it alone and build that feature without looking at any repo.

**The full set is done when:** Claude reads the relevant manuals, receives an app brief like the VidGen one, and builds the backend as fast as the Next.js frontend gets built — without falling on its ass.

---

## One Final Note

The reason this works for Next.js isn't just that the docs exist. It's that the docs were written *for this user's specific stack*. They weren't generic Next.js docs — they were this starter kit's docs.

Do the same thing here. You're not writing generic ADK documentation. You're writing **this user's Python/AI stack documentation** — Vertex AI, Cloud Run, Supabase, Streamlit, FastAPI, GHL CRM. The patterns that appeared in their 10 repos. The gotchas that bit them. The architecture they actually use.

That specificity is what makes it Superman fuel instead of a textbook.

---

_Written by: Claude Code Sonnet 4.6 — 2026-02-24_
_Based on: 10 repo extractions, 35 confirmed patterns, user's production app stack_
_Next step: User will say "please read the mission brief." Then you execute._
