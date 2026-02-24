# Architecture: ADK Wrapper (google-adk-wrapper-v2)

## What This Service Is

The ADK Wrapper is a **stateless FastAPI middleware** that sits between any frontend (Streamlit, Next.js, N8N, mobile) and a Google ADK Agent Bundle (deployed on Cloud Run).

Its sole purpose: convert a simple `{agent_name, message, user_id, session_id?}` request into the correct sequence of ADK API calls, and return a clean `{response, session_id}` result.

---

## System Flow

```
Frontend (Streamlit / Next.js / N8N / Mobile)
    │
    │  POST /run_agent
    │  { agent_name, message, user_id, session_id? }
    ▼
┌─────────────────────────────────────────┐
│         ADK WRAPPER (this service)      │
│         FastAPI + Uvicorn               │
│         Cloud Run (stateless)           │
│                                         │
│  1. Look up agent_url from AGENT_REG.   │
│  2. Create session if none provided     │
│  3. POST /run to ADK bundle             │
│  4. On 404: create new session, retry   │
│  5. Parse events[] → extract final text │
│  6. Return clean response               │
└─────────────────────────────────────────┘
    │
    │  POST /run
    │  { app_name, user_id, session_id, new_message }
    ▼
┌─────────────────────────────────────────┐
│       ADK AGENT BUNDLE (separate svc)   │
│       adk api_server .                  │
│       Cloud Run + Supabase sessions     │
│                                         │
│  - greeting_agent                       │
│  - jarvis_agent                         │
│  - calc_agent                           │
│  - product_agent                        │
│  - ghl_mcp_agent                        │
└─────────────────────────────────────────┘
    │
    │  Response: events[] (array of ADK events)
    ▼
ADK Wrapper extracts final model text
    │
    └── Returns to frontend:
        { response, session_id, agent_name, status }
```

---

## File Structure

```
google-adk-wrapper-v2/
├── main.py              # FastAPI app — all endpoints + helper functions
├── config.py            # Loads config.json → builds AGENT_REGISTRY
├── config.json          # Environment URLs + agent list (active)
├── config-org.json      # Template version (placeholder URLs)
├── main-org.py          # Backup of main.py (identical — preservation pattern)
├── Dockerfile           # Container definition — python:3.12-slim + uvicorn
├── deploy.sh            # gcloud run deploy --source . with env var injection
├── start_cloud.sh       # Local dev helper — sets APP_ENV=cloud, targets live ADK
├── .gcloudignore        # Excludes .venv/, __pycache__, .git, .env, *.json
├── requirements.txt     # fastapi, uvicorn, httpx, pydantic, streamlit (legacy)
└── deploy_docs/         # Human-readable API + deployment reference docs
    ├── overview.md
    ├── api-info.md
    └── deployment.md
```

---

## Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/run_agent` | Run an agent, return response + session_id |
| `POST` | `/get_history` | Fetch + normalize conversation history |
| `GET` | `/health` | Service health check + agent list |
| `GET` | `/agents` | List registered agents |

---

## Configuration Architecture

```
APP_ENV (env var)
    │
    ├── "local"  → adk_bundle_url = http://localhost:8000
    └── "cloud"  → adk_bundle_url = https://[bundle].run.app

config.json
├── environments.local.adk_bundle_url
├── environments.cloud.adk_bundle_url
└── agents: [list of agent names]

config.py → AGENT_REGISTRY = { agent_name: base_url, ... }
```

All agents in a bundle share a single base URL. The registry maps `agent_name → base_url`.

---

## Deployment Architecture

```
Local dev (against local ADK bundle):
    uvicorn main:app --reload --port 8080
    APP_ENV=local (default)

Local dev (against cloud ADK bundle):
    APP_ENV=cloud uvicorn main:app --reload --port 8080
    (see: start_cloud.sh)

Production (Cloud Run):
    Dockerfile: ENV APP_ENV="cloud"
    CMD uvicorn main:app --host 0.0.0.0 --port ${PORT}
    deploy.sh: gcloud run deploy --source . --set-env-vars "ADK_BUNDLE_URL=..."
```

---

## What This Service Does NOT Do

- No state / no database — every request is stateless
- No authentication (currently `--allow-unauthenticated`)
- No agent logic — pure routing + adaptation layer
- No Secret Manager usage (URL is safe to pass as env var, not a secret)
- No GCS usage
- No streaming — waits for full ADK response, then parses

---

## Comparison to n8n-hybrid Repo

In `google-adk-n8n-hybrid-v2`, the docs referenced an "ADK Wrapper" as an external service. **This repo is that service.** The n8n-hybrid session noted "ADK Wrapper implementation — referenced in docs but source isn't in this repo." This repo is the missing piece.

In n8n-hybrid, the call chain was:
```
N8N → [HTTP Request node] → ADK Wrapper ← this repo
                                │
                                └── ADK Bundle (n8n-hybrid repo)
```
