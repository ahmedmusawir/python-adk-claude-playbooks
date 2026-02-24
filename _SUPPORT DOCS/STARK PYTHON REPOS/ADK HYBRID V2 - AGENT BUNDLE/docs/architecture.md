# ADK N8N Hybrid Architecture

**Project:** google-adk-n8n-hybrid-v2
**Extracted:** 2026-02-23

---

## Overview

This is a **production ADK multi-agent bundle** deployed to Google Cloud Run. It exposes 5 specialized agents via the ADK native API, fronted by:
1. An **ADK Wrapper** (FastAPI) — simplifies the complex ADK session API into a single `/run_agent` endpoint
2. An **N8N workflow** — acts as a webhook gateway, normalizes requests, and routes to the wrapper

**Key insight:** ADK's native API requires multi-step session management before agent calls. The FastAPI wrapper hides this complexity so N8N, Streamlit, or any client can call one clean endpoint.

---

## Full System Architecture

```
External Client (browser, app, etc.)
    ↓ POST /webhook/{uuid}
N8N Webhook Node
    ↓
N8N Code Node 1 — normalize payload
    ↓ { agent_name, message, user_id, session_id }
N8N HTTP Request → POST http://localhost:8080/run_agent
    ↓
ADK Wrapper (FastAPI, localhost:8080)
    → Creates/manages ADK sessions automatically
    → Sends message to correct agent
    ↓
ADK Bundle API (Google Cloud Run, port $PORT)
    adk api_server . --session_service_uri=$DB_URI
    ↓
Selected Agent (jarvis_agent / calc_agent / etc.)
    → Fetches instructions from GCS (hot-reload)
    → Calls tools (google_search, calculate, MCPToolset, FunctionTool)
    ↓
Response flows back up the chain
    ↓
N8N Code Node 2 — package response
    ↓ { message, session_id }
N8N Respond to Webhook Node
    ↓
External Client
```

---

## File Structure

```
google-adk-n8n-hybrid-v2/
│
├── Agents (one folder per agent, ADK discovery via __init__.py)
│   ├── greeting_agent/
│   │   ├── agent.py         # root_agent definition
│   │   └── __init__.py      # from .agent import root_agent
│   ├── calc_agent/
│   ├── jarvis_agent/
│   ├── product_agent/
│   └── ghl_mcp_agent/
│
├── utils/
│   ├── gcs_utils.py         # fetch_instructions() — GCS hot-reload pattern
│   └── context_utils.py     # fetch_context(), fetch_document() — knowledge base tools
│
├── docs/                    # documentation
│   ├── overview.md          # user-written overview
│   ├── deployment.md        # user-written deployment guide
│   └── api-info.md          # user-written API reference
│
├── Dockerfile               # shell form CMD for env var substitution
├── Dockerfile.old           # exec form CMD (does NOT work with $PORT)
├── deploy.sh                # gcloud run deploy with secrets injection
├── store_secrets.sh         # .env → Google Secret Manager
├── grant_permissions.sh     # service account IAM setup (one-time)
├── start_server.sh          # local dev server with Supabase session URI
├── .gcloudignore            # CRITICAL: prevents .env/.venv/keys from upload
├── requirements.txt         # full locked dependency set
├── .env_example             # minimal: 2 credentials + 2 GHL values
├── Adk_N8N_Hybrid_v4.json   # N8N workflow export (importable to N8N)
└── cyberize-vertex-api.json # service account key (local dev only, .gcloudignored)
```

---

## Agent Inventory

| Agent | Description | Tools | Instruction Source |
|-------|-------------|-------|-------------------|
| `greeting_agent` | General/intro agent | `FunctionTool(fetch_document)` — reads GCS docs | GCS callable |
| `calc_agent` | Mathematical calculator | custom `calculate()` function | GCS callable |
| `jarvis_agent` | General assistant with web search | `google_search` (built-in) | GCS callable |
| `product_agent` | Product specialist (DockBloxx) | `FunctionTool(fetch_context)` — reads PRODUCTS.md | GCS callable |
| `ghl_mcp_agent` ("Rico") | CRM agent for GoHighLevel | `MCPToolset` → GHL live API | Hardcoded (inline function) |

**Note:** All 5 agents use `gemini-2.5-flash` via Vertex AI. The repo README explicitly says "all other models w/ OpenRouter simply sux!" — vendor consolidation confirmed as a deliberate production decision.

---

## ADK Session Flow (What the Wrapper Abstracts)

The ADK native API requires:
1. POST `/apps/{agent}/users/{user_id}/sessions` — create session
2. POST `/run` — execute agent with session context
3. Parse streaming or final response format

The ADK Wrapper reduces this to:
```
POST /run_agent
{ "agent_name": "...", "message": "...", "user_id": "...", "session_id": "..." }
→ returns { "response": "...", "session_id": "..." }
```

---

## Session Persistence Architecture

```
ADK Bundle (Cloud Run)
    ↓ session_service_uri
Supabase (PostgreSQL)
    - Stores full conversation history
    - Keyed by user_id + session_id
    - Persists across container restarts
    - Survives Cloud Run scale-down
```

**Why Postgres over in-memory:**
- Cloud Run containers can be killed at any time
- In-memory sessions → lost on restart (bad UX)
- Postgres → sessions survive forever
- Supabase = managed Postgres with easy setup

---

## GCS Knowledge Base Architecture

```
GCS Bucket: adk-agent-context-ninth-potion-455712-g9
└── ADK_Agent_Bundle_1/
    ├── greeting_agent/
    │   └── greeting_agent_instructions.txt   ← hot-reload per run
    ├── calc_agent/
    │   └── calc_agent_instructions.txt
    ├── jarvis_agent/
    │   └── jarvis_agent_instructions.txt
    ├── product_agent/
    │   └── product_agent_instructions.txt
    └── context_store/
        ├── PRODUCTS.md                        ← product knowledge base
        └── moose_resume.txt                   ← demo document
```

**Two-tier pattern:**
- `/agent_name/agent_name_instructions.txt` — fetched on every agent run (callable instruction)
- `/context_store/*.md` or `*.txt` — fetched on demand by FunctionTool when agent needs context

---

## Deployment Architecture

**Three-script production pattern:**

```
1. grant_permissions.sh    (ONE-TIME: IAM setup for service account)
    → roles/aiplatform.user
    → roles/storage.objectViewer

2. store_secrets.sh        (WHEN SECRETS CHANGE: .env → Secret Manager)
    .env → reads values → gcloud secrets versions add

3. deploy.sh               (EVERY DEPLOY: source → Cloud Build → Cloud Run)
    gcloud run deploy --source .
    --set-secrets="DB_URI=adk-db-uri:latest,..."
    --set-env-vars="GOOGLE_GENAI_USE_VERTEXAI=TRUE,..."
```

**Source-based deployment:** `--source .` means no local Docker build required. Cloud Build runs in the cloud, using `Dockerfile` to build the image.

---

## Key Architectural Decisions

1. **ADK Wrapper layer** — hides session complexity from upstream callers
2. **N8N as gateway** — workflow automation without custom code; easy to modify flow
3. **Supabase for session state** — cross-restart persistence, no custom session management
4. **GCS callable instructions** — update agent behavior without redeployment
5. **All Vertex (no OpenRouter)** — single-vendor, single auth, consistent behavior
6. **Source-based Cloud Run** — simpler than building Docker locally; Cloud Build handles it

---

_Extracted: 2026-02-23_
