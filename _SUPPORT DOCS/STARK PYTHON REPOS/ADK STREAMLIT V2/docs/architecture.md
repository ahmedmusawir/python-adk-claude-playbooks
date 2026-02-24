# Streamlit Frontend Architecture

**Project:** google-adk-n8n-hybrid-streamlit-v2
**Extracted:** 2026-02-23

---

## What This Repo Is

This is the **Streamlit frontend** for the full ADK N8N Hybrid system. It is the user-facing component in a 4-service architecture. Its job: authenticate users, present a chat interface, route messages to the backend, and persist session bookmarks across logins.

**Repo #6 in the extraction mission.** First pure frontend repo in the series.

---

## System Context (Where This Fits)

```
User Browser
    ↓
Streamlit Frontend (this repo)   ←→   Supabase (auth + session bookmarks)
    ↓                                          ↕
ADK Wrapper (google-adk-wrapper-v2)   GCS (agent instructions)
    ↓                                      ↑ editable from Mission Control
ADK Agent Bundle (google-adk-n8n-hybrid-v2)
    ↓
Google Vertex AI (Gemini models)
```

**This service's role:** UI only. It doesn't talk to ADK directly — everything goes through the ADK Wrapper.

---

## File Structure

```
google-adk-n8n-hybrid-streamlit-v2/
├── chat.py                          # Main entry point: login gate + chat UI
├── chat-org.py                      # V1 backup (direct N8N + direct ADK — see evolution)
├── config.json                      # Environment-aware config + agent list
├── config.py                        # Config loader (APP_ENV-aware, module-level load)
├── Procfile                         # Cloud Run startup: streamlit run chat.py --server.port $PORT
├── deploy.sh                        # One-command Cloud Run deploy
├── grant_streamlit_permissions.sh   # One-time IAM setup for GCS access
├── store_streamlit_secrets.sh       # .env → Secret Manager (run once, or when creds change)
├── start_chat.sh                    # Local dev (no APP_ENV = local mode)
├── start_cloud.sh                   # Hybrid dev: local Streamlit → cloud backend
├── requirements.txt                 # Python dependencies (pinned)
├── .gcloudignore                    # Excludes .venv, .env, *.json (with !config.json exception)
├── .streamlit/
│   └── secrets.toml                 # Local-only secrets (SUPABASE_URL, SUPABASE_KEY)
├── pages/
│   └── 1_Mission_Control.py         # Admin page: edit GCS agent instructions live
└── utils/
    ├── auth.py                      # gatekeeper(), fetch_profile(), save_profile()
    └── gcs_utils.py                 # fetch_instructions(), update_instructions()
```

---

## Application Flow

### Login Flow

```
User visits app
    ↓
chat.py runs (every request = full script execution in Streamlit)
    ↓
Check st.session_state.session
    ├── None → Show login form
    │       ↓ User submits
    │       supabase.auth.sign_in_with_password({email, password})
    │       ↓ Success
    │       st.session_state.session = auth_response.session
    │       st.session_state.agent_sessions = fetch_profile(supabase, user_id)
    │       st.rerun()
    └── Not None → Show main chat app
```

### Agent Switch Flow

```
User selects different agent from dropdown
    ↓
st.session_state.last_selected_agent != selected_agent
    ↓
Load this agent's saved session_id from agent_sessions dict
    ↓
fetch_history(agent_name, user_id, session_id) → POST /get_history to Wrapper
    ↓
st.session_state.messages = history
    ↓
st.rerun() → UI shows history
```

### Message Send Flow

```
User types message → st.chat_input
    ↓
Append user message to st.session_state.messages
    ↓
call_agent_wrapper(agent_name, message, user_id, session_id)
    → POST /run_agent to ADK Wrapper
    → Returns {response, session_id, agent_name, status}
    ↓
If new session_id returned:
    st.session_state.agent_sessions[agent] = new_session_id
    save_profile(supabase, user_id, agent_sessions)  # persist bookmark
    ↓
Append assistant response
    ↓
st.rerun()
```

---

## Two-Page Structure

### Page 1: `chat.py` (Main Chat)
- Authentication gate (login form if not logged in)
- Agent selector dropdown (from config.json)
- Chat history display
- Chat input
- Session bookmark persistence

### Page 2: `pages/1_Mission_Control.py` (Admin Panel)
- Secured by `gatekeeper()` — blocks unauthenticated access
- Fetches current instructions from GCS for each agent
- Text area per agent with editable instructions
- Save button writes new instructions back to GCS
- Changes are live immediately (ADK callable instruction fetches per run)

**Note:** The `1_` prefix in the filename determines sidebar order in Streamlit multi-page apps.

---

## Data Stores

| Store | What It Holds | How Accessed |
|-------|--------------|--------------|
| Supabase Auth | User accounts, authentication | `supabase.auth.sign_in_with_password()` |
| Supabase DB | `adk_n8n_hybrid_profiles` table — `agent_sessions` dict per user | `fetch_profile()` / `save_profile()` |
| GCS | Agent instruction files | `fetch_instructions()` / `update_instructions()` |
| ADK Wrapper | Conversation history (via ADK) | `POST /get_history` |

### `adk_n8n_hybrid_profiles` Table Schema

```
Table: adk_n8n_hybrid_profiles
  id            uuid (primary key = Supabase user ID)
  agent_sessions jsonb  {"greeting_agent": "session-uuid-123", "jarvis_agent": "session-uuid-456"}
```

The `upsert` operation handles first-time users (no row) and returning users (update existing row) with the same code.

---

## Configuration System

### Three Layers of Configuration

**Layer 1: `config.json`** — environment URLs + agent list
```json
{
  "environments": {
    "local":  { "wrapper_url": "http://localhost:8080", ... },
    "cloud":  { "wrapper_url": "https://...run.app", ... }
  },
  "agents": ["greeting_agent", "jarvis_agent", ...]
}
```

**Layer 2: `config.py`** — reads config.json at import time, exports constants
```python
WRAPPER_URL = ...      # used in chat.py
ADK_BUNDLE_URL = ...   # available but not used in chat (only wrapper is needed)
AGENT_OPTIONS = [...]  # populates dropdown
```

**Layer 3: `st.secrets` / Secret Manager** — Supabase credentials only
- Local: `.streamlit/secrets.toml`
- Cloud: `--set-secrets` in deploy.sh injects Secret Manager values

---

## Deployment Architecture

**Service name:** `adk-streamlit-frontend-v2`
**Region:** `us-east1`
**Platform:** Google Cloud Run (source-based deploy)
**Entry point:** `Procfile` (not Dockerfile)

### Key Difference from Other Services

All other services in the ecosystem (ADK Bundle, ADK Wrapper) use a **Dockerfile** for deployment. This Streamlit service uses a **Procfile**.

```
Procfile:
web: streamlit run chat.py --server.port $PORT --server.enableCORS false
```

The `Procfile` is picked up by the Cloud Run buildpack. `$PORT` is resolved by shell (same principle as shell-form Dockerfile CMD in other repos). `--server.enableCORS false` is required for Cloud Run.

---

## The V1 → V2 Evolution

`chat-org.py` preserves V1. Comparing to V2 (`chat.py`) reveals the key simplification:

| Aspect | V1 (chat-org.py) | V2 (chat.py) |
|--------|------------------|--------------|
| Chat routing | `POST` to N8N webhook URL | `POST /run_agent` to ADK Wrapper |
| History fetch | `GET /apps/{agent}/users/{uid}/sessions/{sid}` on ADK directly | `POST /get_history` to ADK Wrapper |
| Response parsing | `json.loads(outer["data"])` — unwrap N8N's double-JSON | `response.json()` — clean JSON from wrapper |
| Agent list | Hardcoded in code | `AGENT_OPTIONS` from config.json |
| Session recovery | None (empty history on stale session) | Wrapper handles 404 auto-recovery |

**Key insight:** The ADK Wrapper (Repo #5) exists to give the frontend this simplified interface. The frontend went from needing to know 3 different service protocols to just one.

---

## Key Architectural Decisions

1. **Proxy through wrapper, not direct** — Frontend doesn't talk to ADK or N8N directly
2. **Supabase for both auth and session bookmarks** — One service, two functions
3. **Procfile over Dockerfile** — Simpler for pure Python apps (no custom build steps)
4. **Per-agent session bookmarks** — Users resume any agent's conversation after login
5. **Admin panel in same app** — Mission Control page reuses auth, shares codebase

---

_Extracted: 2026-02-23_
