# Streamlit + ADK Frontend Integration Guide

**Project:** google-adk-n8n-hybrid-streamlit-v2
**Extracted:** 2026-02-23

---

## Overview

This is a deep-dive reference for building a production Streamlit app that sits in front of an ADK agent system. It covers the complete stack: Supabase auth, per-user session persistence, ADK Wrapper communication, GCS-backed admin panel, and Cloud Run deployment.

---

## The Full Stack

```
Streamlit App (Cloud Run)
    ↓ auth
Supabase
    ├── auth.users (credentials)
    └── adk_n8n_hybrid_profiles (agent_sessions per user)

    ↓ agent calls
ADK Wrapper (Cloud Run)
    → POST /run_agent
    → POST /get_history

    ↓ GCS
Google Cloud Storage (ADK_Agent_Bundle_1/)
    → agent_name/agent_name_instructions.txt (editable from Mission Control)
```

---

## Authentication System

### Supabase Email/Password Auth

The frontend uses Supabase's built-in email/password authentication.

**Flow:**
1. `supabase.auth.sign_in_with_password({"email": ..., "password": ...})`
2. Success → `auth_response.session` contains the session object
3. `st.session_state.session = auth_response.session` — persists across reruns
4. `st.session_state.session.user.id` — the permanent user UUID

**Client setup:**
```python
from supabase import create_client, Client
import streamlit as st

SUPABASE_URL = st.secrets["SUPABASE_URL"]
SUPABASE_KEY = st.secrets["SUPABASE_KEY"]
supabase: Client = create_client(SUPABASE_URL, SUPABASE_KEY)
```

**Auth check:**
```python
if st.session_state.session is None:
    # show login
else:
    # main app
```

### Supabase ANON Key vs Service Role Key

- **ANON key** — Used in frontend. Respects Row Level Security (RLS). Safe to expose in browser.
- **Service role key** — Never use in frontend. Bypasses RLS. Keep server-side only.

The `.streamlit/secrets.toml` uses the ANON key.

### RLS Consideration

The `adk_n8n_hybrid_profiles` table should have RLS enabled:
```sql
-- Allow users to only read/write their own profile
CREATE POLICY "Users can manage own profile" ON adk_n8n_hybrid_profiles
FOR ALL USING (auth.uid() = id);
```

Note: This repo does not show RLS configuration explicitly — it's a gap.

---

## Session Persistence Architecture

### The Problem

ADK sessions expire or reset on server restart. Users need to resume conversations across browser sessions, devices, and days.

### The Solution: Session Bookmarks in Supabase

Every time the backend returns a session_id (new or recovered), the frontend saves it to Supabase under the user's profile.

```python
# Shape of agent_sessions dict:
{
    "greeting_agent": "c4a8f1e2-...",  # ADK session UUID for this user+agent
    "jarvis_agent":   "b7d3c9a1-...",
    "calc_agent":     None              # Not started yet
}
```

**Lifecycle:**
```
First message to greeting_agent:
  session_id = None → Wrapper creates new session → returns new_session_id
  → save to Supabase

Second session (new browser tab):
  Load profile from Supabase → get saved session_id
  → fetch_history(session_id) → resume conversation
```

### What Happens When Session Expires

ADK sessions can expire if the backend restarts (in-memory mode) or hits max age (Supabase mode).

V2 relies on the **ADK Wrapper** for session 404 auto-recovery:
- Wrapper detects 404 from ADK → creates new session → returns new session_id
- Frontend receives the new session_id and overwrites the stale bookmark
- User loses conversation history (session is fresh) but doesn't see an error

**V1 behavior:** Frontend would silently show empty history and continue — no recovery.

---

## Communication with ADK Wrapper

### Endpoint: `POST /run_agent`

**Request:**
```python
{
    "agent_name": "greeting_agent",
    "message": "Hello!",
    "user_id": "supabase-user-uuid",
    "session_id": "adk-session-uuid"  # or None for first message
}
```

**Response:**
```python
{
    "response": "Hello! How can I help you today?",
    "session_id": "adk-session-uuid",  # effective session (may be new)
    "agent_name": "greeting_agent",
    "status": "success"
}
```

**Timeout:** 90 seconds. LLM inference can take 10-30 seconds. Never use default.

### Endpoint: `POST /get_history`

**Request:**
```python
{
    "agent_name": "greeting_agent",
    "user_id": "supabase-user-uuid",
    "session_id": "adk-session-uuid"
}
```

**Response:**
```python
{
    "history": [
        {"role": "user",      "content": "Hello!"},
        {"role": "assistant", "content": "Hello! How can I help?"},
        ...
    ]
}
```

**Returns empty list if:** session_id is None, session not found, or any error.

### Error Handling Pattern

```python
try:
    response = requests.post(url, json=payload, timeout=90)
    response.raise_for_status()
    return response.json()
except Exception as e:
    st.error(f"Failed to connect to Agent Wrapper: {e}")
    return {"response": f"Error: {e}"}
```

Always return a dict with a `"response"` key so the caller doesn't crash on missing key.

---

## Multi-Page App Pattern

### Streamlit Multi-Page Navigation

Files in `pages/` become navigation items in the sidebar automatically.

```
chat.py              → "chat" (main page, no sidebar prefix)
pages/
    1_Mission_Control.py → "Mission Control" (sidebar item 1)
    2_Analytics.py       → "Analytics" (sidebar item 2)
```

**Naming convention:** `{number}_{Page_Name}.py` — the number sets order, underscores become spaces.

### Auth State Across Pages

Each page file is a separate Python script. `st.session_state` IS shared across pages within the same browser session.

```python
# In pages/1_Mission_Control.py:
import streamlit as st
from utils.auth import gatekeeper

gatekeeper()  # Reads st.session_state.session
# If session exists, execution continues normally
```

### The `gatekeeper()` Contract

```python
def gatekeeper():
    """Must be called at the top of every protected page."""
    if 'session' not in st.session_state or st.session_state.session is None:
        st.warning("⚠️ You must be logged in to access this page.")
        st.info("Please log in through the main 'chat' page to continue.")
        st.stop()
```

`st.stop()` = `raise StopException`. Everything below this call does not execute. The user sees the warning/info messages and nothing else.

---

## Cloud Run Deployment

### Service Configuration

```
Service name: adk-streamlit-frontend-v2
Platform:     Cloud Run (fully managed)
Region:       us-east1
Deploy method: --source . (buildpack, not Dockerfile)
Entry point:  Procfile
Auth:         --allow-unauthenticated (Supabase handles app-level auth)
```

### Three-Script Deployment Process

Same pattern as the ADK Bundle and Wrapper:

```bash
# Step 1: One-time IAM setup (run once per project)
./grant_streamlit_permissions.sh
# Grants: roles/storage.objectAdmin to service account

# Step 2: Store secrets (run when Supabase credentials change)
./store_streamlit_secrets.sh
# Reads: .env file
# Stores: supabase-url, supabase-key in Secret Manager

# Step 3: Deploy (run on every code change)
./deploy.sh
# Builds from source, deploys to Cloud Run
```

### `deploy.sh` Breakdown

```bash
gcloud run deploy "adk-streamlit-frontend-v2" \
  --source .                                          # Source-based (Cloud Build)
  --region="us-east1" \
  --project="ninth-potion-455712-g9" \
  --allow-unauthenticated \                           # Public (Supabase handles auth)
  --service-account="stark-vertex-ai@..." \          # For GCS access
  --set-secrets="SUPABASE_URL=supabase-url:latest,SUPABASE_KEY=supabase-key:latest" \
  --set-env-vars="APP_ENV=cloud,WRAPPER_URL=${WRAPPER_URL}"  # Note: WRAPPER_URL env var redundant (see bugs)
```

### Procfile vs Dockerfile

**Streamlit apps use `Procfile`; ADK/FastAPI apps use `Dockerfile`.**

| Aspect | Procfile | Dockerfile |
|--------|---------|------------|
| Complexity | 1 line | 10+ lines |
| Python setup | Buildpack handles | Manual in Dockerfile |
| Use case | Pure Python apps | Custom build steps |
| `$PORT` | Shell expands it | Needs shell form CMD |

**Procfile content:**
```
web: streamlit run chat.py --server.port $PORT --server.enableCORS false
```

### Local Development Modes

```bash
# Mode 1: Fully local (nothing deployed)
streamlit run ./chat.py
# APP_ENV defaults to 'local' → localhost:8080 for wrapper

# Mode 2: Hybrid (local Streamlit, cloud backend)
APP_ENV="cloud" streamlit run ./chat.py
# → Targets live ADK Wrapper on Cloud Run

# Convenience scripts:
./start_chat.sh      # Mode 1
./start_cloud.sh     # Mode 2
```

---

## GCS Integration for Admin Panel

### Bucket Structure (Same as ADK Bundle)

```
gs://adk-agent-context-ninth-potion-455712-g9/
└── ADK_Agent_Bundle_1/
    ├── greeting_agent/
    │   └── greeting_agent_instructions.txt   ← editable from Mission Control
    ├── calc_agent/
    │   └── calc_agent_instructions.txt
    ├── jarvis_agent/
    │   └── jarvis_agent_instructions.txt
    └── product_agent/
        └── product_agent_instructions.txt
```

### Why This Works (The Live Loop)

1. User edits instructions in Mission Control → `update_instructions()` writes to GCS
2. User sends next message → ADK Bundle's callable instruction fetches from GCS
3. Agent uses new instructions — no redeploy needed

**This is the prompt management loop for production agents.**

### Service Account Permissions

The Cloud Run service account (`stark-vertex-ai@ninth-potion-455712-g9.iam.gserviceaccount.com`) needs `roles/storage.objectAdmin`:

```bash
gcloud projects add-iam-policy-binding "ninth-potion-455712-g9" \
  --member="serviceAccount:stark-vertex-ai@..." \
  --role="roles/storage.objectAdmin"
```

---

## Data Flow Diagram: Full Round Trip

```
User types message
    ↓
st.chat_input captures prompt
    ↓
Immediately append to st.session_state.messages + display
    ↓
call_agent_wrapper(agent_name, message, user_id, session_id)
    ↓ POST /run_agent (90s timeout)
ADK Wrapper
    ↓
ADK Bundle → Vertex AI (Gemini)
    ↑
Response flows back
    ↓
response_data = {response, session_id, agent_name, status}
    ↓
If new session_id:
    Update st.session_state.agent_sessions
    save_profile(supabase, user_id, agent_sessions)  ← Supabase upsert
    ↓
Append assistant response to messages
    ↓
st.rerun()  ← script re-executes from top, displays new message
```

---

## Known Issues / Bugs Found

1. **`utils/gcs_utils.py` hardcodes `BUCKET_NAME`** — Should use `os.getenv("GCS_BUCKET_NAME")`. Same bug seen in `google-adk-n8n-hybrid-v2`.

2. **`deploy.sh` sets `WRAPPER_URL` as env var but nothing reads it** — `config.py` reads from `config.json`, not from env vars. The `--set-env-vars="WRAPPER_URL=..."` line is redundant. If you want the URL to be env-var-configurable, `config.py` would need to be updated to check `os.getenv("WRAPPER_URL")` first.

3. **`cyberize-vertex-api.json` exists in repo root** — A service account JSON key in the git repo. The `.gcloudignore` prevents it from deploying, but it should be in `.gitignore` as well and rotated.

4. **`requirements.txt` is extremely heavy** — Contains `google-adk`, `litellm`, `fastapi`, `uvicorn`, and many other backend-only dependencies. This Streamlit app only needs: `streamlit`, `supabase`, `requests`, `google-cloud-storage`. The oversized requirements.txt increases Cloud Build time significantly.

5. **No RLS on `adk_n8n_hybrid_profiles`** — Not visible in the code, but if not configured, any authenticated user could read any other user's sessions via the Supabase API.

6. **Mission Control has no role-based access control** — Any authenticated user can edit any agent's instructions. Fine for single-team use, would need role checks for multi-tenant.

---

## Comparison to V1 (`chat-org.py`)

| Concern | V1 | V2 |
|---------|----|----|
| Chat routing | Direct to N8N webhook | Via ADK Wrapper `/run_agent` |
| History fetch | Direct to ADK `/apps/{agent}/...` | Via ADK Wrapper `/get_history` |
| Session recovery | None | Wrapper handles 404 auto-recovery |
| Response parsing | `json.loads(outer["data"])` (double-JSON) | `response.json()` (clean JSON) |
| Agent list | Hardcoded: `["greeting_agent", "calc_agent", ...]` | From `config.json["agents"]` |
| URLs | Hardcoded in code | From `config.py` (env-aware) |

**V2 is significantly simpler** because the ADK Wrapper (Repo #5) absorbed all the complexity.

---

## Key Takeaways for Manuals

1. **Streamlit multi-page auth needs `gatekeeper()`** — One function, call at top of every page
2. **Supabase does both auth + profile** — `auth.sign_in_with_password` + `table().upsert()`
3. **`st.secrets` is the credential pattern** — Works identically locally and in Cloud Run
4. **`Procfile` over Dockerfile for Streamlit** — Simpler, fewer lines, same result
5. **Frontend delegates everything to the wrapper** — It should not know about N8N, ADK session URLs, or event parsing
6. **Session bookmarks enable persistent conversations** — Store `{agent: session_id}` per user in DB
7. **Mission Control = admin panel** — Same auth, same codebase, live prompt editing via GCS
8. **Three scripts for Cloud Run lifecycle** — grant permissions → store secrets → deploy

---

_Extracted: 2026-02-23_
