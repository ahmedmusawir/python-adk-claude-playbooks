# Streamlit Frontend Patterns

**Project:** google-adk-n8n-hybrid-streamlit-v2
**Extracted:** 2026-02-23

---

## Overview

These patterns are copy-pasteable building blocks for Streamlit apps with authentication, multi-page navigation, Supabase integration, and Cloud Run deployment.

---

## Pattern 1: Supabase Auth Gate (Login Wall)

The entire app is gated behind a login form. One `if/else` in the main script.

```python
import streamlit as st
from supabase import create_client, Client

SUPABASE_URL = st.secrets["SUPABASE_URL"]
SUPABASE_KEY = st.secrets["SUPABASE_KEY"]
supabase: Client = create_client(SUPABASE_URL, SUPABASE_KEY)

# Initialize session state
if 'session' not in st.session_state:
    st.session_state.session = None

# Login gate
if st.session_state.session is None:
    st.title("Login")
    with st.form("login_form"):
        email = st.text_input("Email")
        password = st.text_input("Password", type="password")
        submitted = st.form_submit_button("Login")

    if submitted:
        try:
            auth_response = supabase.auth.sign_in_with_password({
                "email": email, "password": password
            })
            st.session_state.session = auth_response.session
            st.rerun()
        except Exception as e:
            st.error(f"Authentication failed: {e}")

else:
    # Main app goes here
    user_id = st.session_state.session.user.id
    # ... rest of app
```

**Key points:**
- Check `st.session_state.session is None` — not just `not session`
- Use `supabase.auth.sign_in_with_password()` for email/password
- Always call `st.rerun()` after login to refresh the whole script
- `user_id = st.session_state.session.user.id` — permanent user identifier

---

## Pattern 2: `gatekeeper()` — Multi-Page Auth Guard

For any page in `pages/`, call this one function at the top to block unauthenticated access.

```python
# utils/auth.py
import streamlit as st

def gatekeeper():
    """Checks for a valid session and stops the page if not logged in."""
    if 'session' not in st.session_state or st.session_state.session is None:
        st.warning("⚠️ You must be logged in to access this page.")
        st.info("Please log in through the main page first.")
        st.stop()  # Halts all execution — nothing below runs
```

**Usage in any page file:**
```python
# pages/1_Admin.py
from utils.auth import gatekeeper

gatekeeper()  # One line secures the entire page

# Rest of page code only runs if logged in
st.title("Admin Panel")
```

**Why `st.stop()` works:** Streamlit executes page files as scripts. `st.stop()` raises `StopException` which halts execution cleanly without an error display.

---

## Pattern 3: Supabase Profile — Fetch + Save

Store and retrieve per-user data (any JSON blob) in Supabase.

```python
# utils/auth.py
from supabase import Client

def fetch_profile(supabase: Client, user_id: str) -> dict:
    """Fetches user profile data from Supabase."""
    try:
        response = (supabase.table("your_profiles_table")
                           .select("data_column")
                           .eq("id", user_id)
                           .execute())
        if response.data:
            return response.data[0].get("data_column", {})
        return {}  # New user — no profile yet
    except Exception as e:
        st.error(f"Error fetching profile: {e}")
        return {}

def save_profile(supabase: Client, user_id: str, data: dict):
    """Saves user profile data to Supabase (upsert = insert or update)."""
    try:
        supabase.table("your_profiles_table").upsert({
            "id": user_id,
            "data_column": data
        }).execute()
    except Exception as e:
        st.error(f"Error saving profile: {e}")
```

**`upsert` is the key:** Handles both new users (no row) and returning users (update row) with one call. Use `id` as the primary key — it's the Supabase `auth.users` UUID.

**Load on login:**
```python
st.session_state.agent_sessions = fetch_profile(supabase, user_id)
```

**Save on state change:**
```python
st.session_state.agent_sessions[selected_agent] = new_session_id
save_profile(supabase, user_id, st.session_state.agent_sessions)
```

---

## Pattern 4: Per-Agent Session Bookmark

Track which ADK session each user has for each agent. Resume on next login.

```python
# State shape in st.session_state.agent_sessions:
# {
#   "greeting_agent": "uuid-session-123",
#   "jarvis_agent":   "uuid-session-456",
# }

# On agent selection change:
if st.session_state.get("last_selected_agent") != selected_agent:
    st.session_state.last_selected_agent = selected_agent

    # Get saved session ID for this agent (None if first time)
    resumed_session_id = st.session_state.agent_sessions.get(selected_agent)

    # Load history for this session
    history = fetch_history(selected_agent, user_id, resumed_session_id)
    st.session_state.messages = history
    st.rerun()

# On new session created by backend:
new_session_id = response_data.get("session_id")
if new_session_id and new_session_id != current_session_id:
    st.session_state.agent_sessions[selected_agent] = new_session_id
    save_profile(supabase, user_id, st.session_state.agent_sessions)
```

---

## Pattern 5: Chat UI (Standard Streamlit Chat)

```python
# Display existing messages
for message in st.session_state.get("messages", []):
    with st.chat_message(message["role"]):
        st.markdown(message["content"])

# Handle new input
if prompt := st.chat_input(f"Ask {selected_agent} a question..."):
    # Immediately show user message
    st.session_state.messages.append({"role": "user", "content": prompt})
    with st.chat_message("user"):
        st.markdown(prompt)

    # Call backend
    with st.spinner("Agent is thinking..."):
        response_data = call_agent_wrapper(...)

    # Show agent response
    assistant_response = response_data.get("response", "Error")
    st.session_state.messages.append({"role": "assistant", "content": assistant_response})

    st.rerun()
```

**Message format:** `{"role": "user" | "assistant", "content": "..."}`

---

## Pattern 6: Call ADK Wrapper (`/run_agent`)

```python
import requests

def call_agent_wrapper(agent_name, message, user_id, session_id):
    """Send message to agent via ADK Wrapper. Returns response + effective session_id."""
    payload = {
        "agent_name": agent_name,
        "message": message,
        "user_id": user_id,
        "session_id": session_id  # None on first call — wrapper creates one
    }
    try:
        response = requests.post(f"{WRAPPER_URL}/run_agent", json=payload, timeout=90)
        response.raise_for_status()
        return response.json()
        # Returns: {"response": "...", "session_id": "...", "agent_name": "...", "status": "..."}
    except Exception as e:
        st.error(f"Failed to connect to Agent Wrapper: {e}")
        return {"response": f"Error: {e}"}
```

**Timeout is 90 seconds** — LLM calls can be slow. Don't use default (never).

---

## Pattern 7: Fetch History via Wrapper (`/get_history`)

```python
def fetch_history(agent_name, user_id, session_id):
    """Load conversation history for a session via ADK Wrapper."""
    if not session_id or not WRAPPER_URL:
        return []
    try:
        payload = {
            "agent_name": agent_name,
            "user_id": user_id,
            "session_id": session_id
        }
        response = requests.post(f"{WRAPPER_URL}/get_history", json=payload, timeout=30)
        response.raise_for_status()
        return response.json().get("history", [])
        # Returns: [{"role": "user"|"assistant", "content": "..."}]
    except Exception as e:
        st.error(f"Failed to fetch history: {e}")
        return []
```

---

## Pattern 8: GCS Read/Write (Agent Instructions)

```python
# utils/gcs_utils.py
from google.cloud import storage

BUCKET_NAME = "your-bucket-name"
BASE_FOLDER = "ADK_Agent_Bundle_1"

def fetch_instructions(agent_name: str) -> str:
    """Read agent instructions from GCS."""
    try:
        client = storage.Client()
        bucket = client.bucket(BUCKET_NAME)
        blob = bucket.blob(f"{BASE_FOLDER}/{agent_name}/{agent_name}_instructions.txt")
        return blob.download_as_text(encoding='utf-8')
    except Exception as e:
        print(f"ERROR fetching instructions for '{agent_name}': {e}")
        return f"Error: Could not load instructions for {agent_name}."

def update_instructions(agent_name: str, new_content: str):
    """Write new agent instructions to GCS."""
    try:
        client = storage.Client()
        bucket = client.bucket(BUCKET_NAME)
        blob = bucket.blob(f"{BASE_FOLDER}/{agent_name}/{agent_name}_instructions.txt")
        blob.upload_from_string(new_content, content_type='text/plain')
    except Exception as e:
        print(f"ERROR updating instructions for '{agent_name}': {e}")
        raise  # Re-raise so caller can handle (e.g., show error in UI)
```

**Authentication:** In Cloud Run, `storage.Client()` uses ADC automatically. In local dev, use `gcloud auth application-default login`. No key management needed.

---

## Pattern 9: Mission Control Admin Page

A self-contained admin page for editing GCS agent instructions from the browser.

```python
# pages/1_Mission_Control.py
import streamlit as st
from utils.gcs_utils import fetch_instructions, update_instructions
from utils.auth import gatekeeper

gatekeeper()  # Secure the page

st.set_page_config(page_title="Mission Control", page_icon="🎛️")
st.title("🎛️ Mission Control")
st.markdown("Update agent instructions in real-time.")

AGENT_NAMES = ["greeting_agent", "calc_agent", "jarvis_agent", "product_agent"]

for agent in AGENT_NAMES:
    st.divider()
    st.subheader(f"Instructions for: `{agent}`")

    current_instructions = fetch_instructions(agent)

    new_instructions = st.text_area(
        label="Modify instructions:",
        value=current_instructions,
        height=250,
        key=f"{agent}_textarea"  # Unique key required for each widget
    )

    if st.button(f"Save for {agent}", key=f"{agent}_button"):
        try:
            update_instructions(agent, new_instructions)
            st.toast(f"✅ Instructions for `{agent}` updated.", icon="✅")
        except Exception as e:
            st.error(f"Failed to update: {e}")
```

**Key:** `key=f"{agent}_textarea"` — unique key required when creating multiple widgets of the same type in a loop.

---

## Pattern 10: Config.py Module-Level Load

Load config once at import time, export as module-level constants.

```python
# config.py
import os, json

def load_config():
    env = os.getenv("APP_ENV", "local").lower()
    with open('config.json', 'r') as f:
        config_data = json.load(f)
    env_config = config_data.get("environments", {}).get(env)
    if not env_config:
        raise ValueError(f"Config for environment '{env}' not found.")
    return {
        "wrapper_url": env_config.get("wrapper_url"),
        "agent_options": config_data.get("agents", [])
    }

# Load once at import time
_config = load_config()
WRAPPER_URL = _config["wrapper_url"]
AGENT_OPTIONS = _config["agent_options"]

print(f"✅ Config loaded for [{os.getenv('APP_ENV', 'local').upper()}] mode.")
```

**Import in app:**
```python
from config import WRAPPER_URL, AGENT_OPTIONS
```

---

## Pattern 11: Procfile for Streamlit on Cloud Run

```
web: streamlit run chat.py --server.port $PORT --server.enableCORS false
```

- `$PORT` — Cloud Run injects the port. Shell-form resolves this variable.
- `--server.enableCORS false` — Required for Cloud Run (no custom CORS headers needed).
- No `--server.headless true` needed (Cloud Run sets this automatically via env).

**Start scripts for different modes:**
```bash
# start_chat.sh — local dev (uses local URLs)
streamlit run ./chat.py --logger.level error

# start_cloud.sh — hybrid dev (local Streamlit → cloud backend)
APP_ENV="cloud" streamlit run ./chat.py --logger.level error
```

---

## Pattern 12: `.gcloudignore` with Config Exception

Standard pattern for Streamlit apps on Cloud Run:

```
# Ignore virtualenv (gigabytes)
.venv/

# Ignore Python cache
__pycache__/
*.pyc

# Ignore git directory and local env
.git/
.gitignore
.env

# Ignore logs and service account keys
*.log
*.json

# BUT include the app's config file
!config.json
```

**The `!config.json` exception** is critical — `config.json` is not a secret (no credentials in it), but `*.json` would otherwise exclude it along with service account keys.

---

## Pattern 13: `st.secrets` → Secret Manager Dual Path

Same code works locally and in Cloud Run:

```python
# In app code (works everywhere):
SUPABASE_URL = st.secrets["SUPABASE_URL"]
SUPABASE_KEY = st.secrets["SUPABASE_KEY"]
```

**Local:** `.streamlit/secrets.toml`
```toml
SUPABASE_URL = "https://your-project.supabase.co"
SUPABASE_KEY = "your-anon-key"
```

**Cloud Run:** `--set-secrets` in deploy.sh injects Secret Manager values as env vars:
```bash
--set-secrets="SUPABASE_URL=supabase-url:latest,SUPABASE_KEY=supabase-key:latest"
```

Streamlit reads `st.secrets` from env vars in Cloud Run automatically — no code change needed.

---

## Pattern 14: Logout

```python
st.sidebar.write(f"Authenticated as: {st.session_state.session.user.email}")
if st.sidebar.button("Logout"):
    st.session_state.session = None
    st.rerun()
```

Setting `session = None` returns the user to the login form on next rerun.

---

## Pattern 15: Agent Dropdown from Config

```python
# In sidebar:
from config import AGENT_OPTIONS

selected_agent = st.sidebar.selectbox("Choose an agent:", options=AGENT_OPTIONS)
st.sidebar.info(f"Chatting with: **{selected_agent}**")
```

Agent list comes from `config.json["agents"]` — not hardcoded. Add new agents to the config, they appear in all UIs automatically.

---

## Patterns Summary

| Pattern | Purpose |
|---------|---------|
| Supabase auth gate | Login wall for entire app |
| `gatekeeper()` | Per-page auth guard (multi-page apps) |
| `fetch_profile` / `save_profile` | User data persistence (any JSON blob) |
| Per-agent session bookmarks | Resume agent conversations across logins |
| Standard chat UI | `st.chat_message` + `st.chat_input` |
| `call_agent_wrapper()` | Route messages through ADK Wrapper |
| `fetch_history()` | Load conversation history |
| GCS read/write | Fetch/update live agent instructions |
| Mission Control page | Admin panel for prompt management |
| Module-level config load | `config.py` pattern for env-aware settings |
| `Procfile` | Streamlit Cloud Run entry point |
| `.gcloudignore` with exception | Secure source deploy |
| `st.secrets` dual path | Same code for local + cloud secrets |
| Logout | Clear session state, rerun |
| Agent dropdown from config | Dynamic agent list from JSON |

---

_Extracted: 2026-02-23_
