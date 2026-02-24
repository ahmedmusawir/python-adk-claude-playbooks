# Streamlit Patterns

## What This Is For

Complete UI pattern library for building Streamlit frontends on top of ADK agents. Covers auth, session management, chat UI, live preview, undo stacks, and Cloud Run deployment.

Every pattern here is sourced from production apps: ADK Streamlit V2 (GHL CRM agent frontend) and GHL HTML Email Template Generator.

---

## Quick Reference — Two-Screen App Shell

Every Streamlit AI app in this stack uses a two-screen pattern: a setup/auth screen and a main app screen. Single file, no routing.

```python
# app.py
import streamlit as st
from config import AGENTS, DEFAULT_AGENT

def main():
    st.set_page_config(page_title="My Agent App", layout="wide")
    init_session_state()

    # Screen 1: Auth gate
    if st.session_state.get("supabase_session") is None:
        show_login()
        return  # Stop here — don't render main app

    # Screen 2: Main app
    show_main_app()


def init_session_state():
    """Initialize all session state keys upfront. Prevents KeyError on first run."""
    defaults = {
        "supabase_session": None,
        "supabase_user": None,
        "agent_session_id": None,
        "selected_agent": DEFAULT_AGENT,
        "messages": [],
        "history": [],      # For undo stack
    }
    for key, val in defaults.items():
        if key not in st.session_state:
            st.session_state[key] = val
```

---

## Pattern 1: Supabase Auth Gate

Login form that blocks the app until the user authenticates. Minimal — no routing, just conditional rendering.

```python
from supabase import create_client
import streamlit as st
import os

def get_supabase():
    url = get_secret("SUPABASE_URL")       # See Pattern 3 for get_secret()
    key = get_secret("SUPABASE_ANON_KEY")
    return create_client(url, key)

def show_login():
    st.title("Sign In")
    email = st.text_input("Email")
    password = st.text_input("Password", type="password")

    if st.button("Sign In"):
        try:
            supabase = get_supabase()
            result = supabase.auth.sign_in_with_password({
                "email": email,
                "password": password
            })
            st.session_state["supabase_session"] = result.session
            st.session_state["supabase_user"] = result.user
            st.rerun()  # ← triggers re-render, now passes auth gate
        except Exception as e:
            st.error(f"Login failed: {e}")
```

---

## Pattern 2: gatekeeper() — Multi-Page Protection

For multi-page Streamlit apps, call `gatekeeper()` at the top of every page that requires auth. One call = page stops rendering if not authenticated.

```python
# utils/auth.py
import streamlit as st

def gatekeeper():
    """Call at top of any page requiring auth. Stops page render if not authenticated."""
    if st.session_state.get("supabase_session") is None:
        st.warning("Please sign in to access this page.")
        st.stop()  # ← halts rendering of the rest of the page
```

Usage in a page file:
```python
# pages/02_Chat.py
from utils.auth import gatekeeper
gatekeeper()

# Everything below only runs if authenticated
st.title("Chat with Agent")
```

---

## Pattern 3: st.secrets → Secret Manager Dual Path

Use `st.secrets` locally (via `.streamlit/secrets.toml`). In Cloud Run, fall back to env vars injected from Secret Manager. No code changes between environments.

```python
import os
import streamlit as st

def get_secret(key: str) -> str:
    """Read from st.secrets (local) or env var (Cloud Run)."""
    try:
        return st.secrets[key]
    except (KeyError, FileNotFoundError):
        val = os.environ.get(key)
        if not val:
            raise ValueError(f"Missing secret: {key}")
        return val

# Usage:
SUPABASE_URL = get_secret("SUPABASE_URL")
SUPABASE_KEY = get_secret("SUPABASE_ANON_KEY")
ADK_WRAPPER_URL = get_secret("ADK_WRAPPER_URL")
```

Local `.streamlit/secrets.toml`:
```toml
SUPABASE_URL = "https://xxx.supabase.co"
SUPABASE_ANON_KEY = "eyJ..."
ADK_WRAPPER_URL = "https://your-wrapper.run.app"
```

Cloud Run: inject via `--set-secrets=SUPABASE_URL=supabase-url:latest`.

---

## Pattern 4: Per-User Session Bookmarks

Each user gets their own mapping of `{agent_name: session_id}` stored in Supabase. When they return, they resume the same conversation. Works across devices.

```python
import json
import streamlit as st

TABLE = "user_profiles"

def load_session_bookmarks(user_id: str) -> dict:
    """Load {agent_name: session_id} map for this user."""
    supabase = get_supabase()
    result = supabase.table(TABLE).select("session_bookmarks").eq("user_id", user_id).execute()
    if result.data:
        raw = result.data[0].get("session_bookmarks", "{}")
        return json.loads(raw) if isinstance(raw, str) else raw
    return {}

def save_session_bookmark(user_id: str, agent_name: str, session_id: str) -> None:
    """Upsert session_id for this agent into user's bookmark map."""
    supabase = get_supabase()
    bookmarks = load_session_bookmarks(user_id)
    bookmarks[agent_name] = session_id

    supabase.table(TABLE).upsert({
        "user_id": user_id,
        "session_bookmarks": json.dumps(bookmarks),
    }).execute()

def get_or_create_agent_session(user_id: str, agent_name: str) -> str:
    """Return existing session_id for this user+agent, or create new one."""
    bookmarks = load_session_bookmarks(user_id)
    if agent_name in bookmarks:
        return bookmarks[agent_name]

    # New session
    import uuid
    session_id = f"{agent_name}-{user_id[:8]}-{uuid.uuid4().hex[:8]}"
    save_session_bookmark(user_id, agent_name, session_id)
    return session_id
```

At app startup:
```python
if st.session_state.get("agent_session_id") is None:
    user_id = st.session_state["supabase_user"].id
    agent_name = st.session_state["selected_agent"]
    st.session_state["agent_session_id"] = get_or_create_agent_session(user_id, agent_name)
```

---

## Pattern 5: Chat UI — Sidebar + Main Preview Layout

Standard layout for agent apps with a content output (HTML, document, analysis). Chat in sidebar, output in main panel.

```python
def show_main_app():
    # Sidebar: chat
    with st.sidebar:
        st.title("Chat")
        show_chat_sidebar()

    # Main panel: content preview
    show_content_preview()


def show_chat_sidebar():
    # Scrollable chat history container
    chat_container = st.container(height=500)
    with chat_container:
        for msg in st.session_state["messages"]:
            with st.chat_message(msg["role"]):
                st.markdown(msg["content"])

    # Input at bottom
    if user_input := st.chat_input("Type a message..."):
        handle_user_input(user_input)


def handle_user_input(user_input: str):
    # Add to UI immediately
    st.session_state["messages"].append({"role": "user", "content": user_input})

    with st.spinner("Thinking..."):
        response = call_agent(user_input, st.session_state["agent_session_id"])

    st.session_state["messages"].append({"role": "assistant", "content": response})
    st.rerun()
```

---

## Pattern 6: Call Agent — Via ADK Wrapper

Never call ADK directly from Streamlit. Always go through the FastAPI wrapper. Simpler, handles session management.

```python
import httpx
import streamlit as st

def call_agent(message: str, session_id: str) -> str:
    """POST to ADK Wrapper /run_agent endpoint."""
    wrapper_url = get_secret("ADK_WRAPPER_URL")
    agent_name = st.session_state["selected_agent"]

    try:
        resp = httpx.post(
            f"{wrapper_url}/run_agent",
            json={
                "agent_name": agent_name,
                "message": message,
                "user_id": st.session_state["supabase_user"].id,
                "session_id": session_id,
            },
            timeout=60.0,
        )
        resp.raise_for_status()
        data = resp.json()

        # Save returned session_id (may differ from what we sent)
        if "session_id" in data:
            st.session_state["agent_session_id"] = data["session_id"]

        return data.get("response", "")
    except httpx.TimeoutException:
        return "Request timed out. Please try again."
    except Exception as e:
        return f"Error: {e}"
```

---

## Pattern 7: Call Agent — Direct ADK (No Wrapper)

When ADK runner is in the same process (local agent, not deployed separately):

```python
import asyncio
from google.genai import types
from my_agent.agent import runner, session_service, APP_NAME

USER_ID = "streamlit_user"

def call_agent_direct(user_input: str, session_id: str) -> str:
    async def _run():
        # Ensure session exists
        session = session_service.get_session(
            app_name=APP_NAME, user_id=USER_ID, session_id=session_id
        )
        if session is None:
            session_service.create_session(
                app_name=APP_NAME, user_id=USER_ID, session_id=session_id
            )

        content = types.Content(
            role="user",
            parts=[types.Part.from_text(text=user_input)]
        )

        response = ""
        async for event in runner.run_async(
            user_id=USER_ID,
            session_id=session_id,
            new_message=content,
        ):
            if event.get_function_calls():
                for fn_call in event.get_function_calls():
                    st.toast(f"⚙️ {fn_call.name}", icon="🔧")
            if event.text:
                response += event.text

        return response

    return asyncio.run(_run())
```

---

## Pattern 8: Agent Session ID in Session State (Amnesia Prevention)

The ADK agent's session ID must live in `st.session_state`. If it's a local variable, every Streamlit rerun creates a new session and the agent loses conversation history.

```python
def init_session_state():
    if "agent_session_id" not in st.session_state:
        st.session_state["agent_session_id"] = None  # Will be set after auth

# After auth:
if st.session_state["agent_session_id"] is None:
    st.session_state["agent_session_id"] = get_or_create_agent_session(
        user_id, agent_name
    )

# Always use from session state:
session_id = st.session_state["agent_session_id"]
response = call_agent(user_input, session_id)
```

---

## Pattern 9: Undo Stack

Saves content state before each agent call. Lets the user roll back to a previous version.

```python
def save_state(content: str) -> None:
    """Push current content onto history stack."""
    st.session_state["history"].append(content)

def undo() -> str | None:
    """Pop last state from history stack. Returns None if empty."""
    if st.session_state["history"]:
        return st.session_state["history"].pop()
    return None

# In UI:
col1, col2 = st.columns([1, 1])
with col1:
    if st.button("↩️ Undo"):
        previous = undo()
        if previous:
            st.session_state["current_content"] = previous
            st.rerun()
        else:
            st.warning("Nothing to undo.")
```

Before each agent call:
```python
def handle_user_input(user_input: str):
    save_state(st.session_state["current_content"])  # Save before mutating
    response = call_agent(user_input, st.session_state["agent_session_id"])
    st.session_state["current_content"] = response
    st.rerun()
```

---

## Pattern 10: Live HTML Preview

Display HTML content that updates as the agent edits it.

```python
import streamlit.components.v1 as components

def show_content_preview():
    html_content = st.session_state.get("current_html", "")

    if html_content:
        components.html(html_content, height=600, scrolling=True)
    else:
        st.info("Start chatting to generate content.")
```

---

## Pattern 11: st.toast() for Non-Blocking Tool Feedback

Surface tool execution to the user without blocking the UI or interrupting the stream.

```python
async for event in runner.run_async(...):
    if event.get_function_calls():
        for fn_call in event.get_function_calls():
            st.toast(f"Running: {fn_call.name}", icon="⚙️")
    if event.text:
        response_text += event.text
```

---

## Pattern 12: Mission Control — Live GCS Prompt Editing

Admin panel that lets non-technical users edit agent system prompts from the browser. Changes take effect immediately — no redeploy.

```python
# pages/99_Mission_Control.py
import streamlit as st
from utils.auth import gatekeeper
from google.cloud import storage

gatekeeper()  # Auth required

BUCKET = "your-gcs-bucket"
PROMPTS = {
    "search_agent": "prompts/search_agent.md",
    "email_agent": "prompts/email_agent.md",
}

def read_gcs(path: str) -> str:
    client = storage.Client()
    bucket = client.bucket(BUCKET)
    blob = bucket.blob(path)
    return blob.download_as_text()

def write_gcs(path: str, content: str) -> None:
    client = storage.Client()
    bucket = client.bucket(BUCKET)
    blob = bucket.blob(path)
    blob.upload_from_string(content)

st.title("Mission Control")
agent = st.selectbox("Select Agent", list(PROMPTS.keys()))
prompt_path = PROMPTS[agent]

current_prompt = read_gcs(prompt_path)
new_prompt = st.text_area("System Prompt", value=current_prompt, height=400)

if st.button("Save Prompt"):
    write_gcs(prompt_path, new_prompt)
    st.success("Saved. Agent will use new prompt on next request.")
```

In your agent, use a callable instruction to fetch from GCS at request time:
```python
from google.cloud import storage

def get_instruction() -> str:
    client = storage.Client()
    bucket = client.bucket("your-gcs-bucket")
    blob = bucket.blob("prompts/my_agent.md")
    return blob.download_as_text()

root_agent = Agent(
    name="my_agent",
    model="gemini-2.5-flash",
    description="Agent with hot-reloadable prompts.",
    instruction=get_instruction,  # ← callable, not get_instruction()
    tools=[],
)
```

---

## Pattern 13: Logout

```python
def logout():
    try:
        get_supabase().auth.sign_out()
    except Exception:
        pass
    # Clear all session state
    for key in list(st.session_state.keys()):
        del st.session_state[key]
    st.rerun()

# In sidebar:
if st.sidebar.button("Sign Out"):
    logout()
```

---

## Pattern 14: Agent Dropdown from Config

When the app supports multiple agents, let users switch between them. Changing agent loads the appropriate session bookmark.

```python
# config.py
AGENTS = {
    "search_agent": "Research anything with live search",
    "email_agent": "Generate and edit email templates",
    "crm_agent": "Interact with GHL CRM",
}
DEFAULT_AGENT = "search_agent"
```

```python
# In sidebar:
agent_options = list(AGENTS.keys())
selected = st.sidebar.selectbox(
    "Agent",
    agent_options,
    index=agent_options.index(st.session_state["selected_agent"]),
    format_func=lambda k: k.replace("_", " ").title(),
)

if selected != st.session_state["selected_agent"]:
    st.session_state["selected_agent"] = selected
    st.session_state["messages"] = []
    # Load session bookmark for new agent
    user_id = st.session_state["supabase_user"].id
    st.session_state["agent_session_id"] = get_or_create_agent_session(user_id, selected)
    st.rerun()

st.sidebar.caption(AGENTS[selected])
```

---

## Pattern 15: Full-Width Chat (No Sidebar)

When the app is chat-only with no preview panel, use a full-width layout instead of the sidebar pattern from Pattern 5.

```python
def show_main_app():
    st.title("Chat with Agent")

    # Scrollable history in main area
    chat_box = st.container(height=600)
    with chat_box:
        for msg in st.session_state["messages"]:
            with st.chat_message(msg["role"]):
                st.markdown(msg["content"])

    # Input at bottom of page, always visible
    if prompt := st.chat_input("Message..."):
        handle_user_input(prompt)
```

Use this for simple Q&A agents. Use Pattern 5 (sidebar chat + main preview) when the agent produces visual output (HTML, documents, analysis).

---

## Cloud Run Deployment

**Procfile** — simpler than Dockerfile for pure Streamlit apps:
```
web: streamlit run app.py --server.port $PORT --server.address 0.0.0.0
```

**Dockerfile** — when you need more control:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD streamlit run app.py --server.port $PORT --server.address 0.0.0.0
```

**`.gcloudignore`** — exclude heavy local-only folders, but keep `.streamlit/`:
```
.git
.venv
__pycache__
*.pyc
node_modules
.streamlit/secrets.toml
# Note: do NOT ignore .streamlit/ itself — config.toml is needed at runtime
```

**Deploy:**
```bash
gcloud run deploy my-streamlit-app \
  --source . \
  --region us-central1 \
  --allow-unauthenticated \
  --set-secrets=SUPABASE_URL=supabase-url:latest \
  --set-secrets=SUPABASE_ANON_KEY=supabase-anon-key:latest \
  --set-secrets=ADK_WRAPPER_URL=adk-wrapper-url:latest \
  --min-instances=1
```

---

## Gotchas

**1. agent_session_id must be in st.session_state**
Streamlit reruns the entire script on every user interaction. A local variable for session ID gets lost. Put it in `st.session_state` or the agent starts a new conversation every message.

**2. init_session_state() before any conditional rendering**
If you check `st.session_state["supabase_session"]` before initializing it, you get a `KeyError`. Always call `init_session_state()` first thing in `main()`.

**3. asyncio.run() works in Streamlit**
Streamlit is synchronous. `asyncio.run()` creates its own event loop and works fine. The common mistake is putting it inside an `async` function — don't. Use `asyncio.run()` from synchronous Streamlit callbacks.

**4. st.container(height=N) for scrollable chat**
Without a height constraint, the chat history expands the page infinitely and pushes the input off screen. Always use `height=` parameter.

**5. st.rerun() after every state change**
Streamlit only re-renders on user interaction by default. After an agent response, call `st.rerun()` to update the chat history display.

**6. .streamlit/secrets.toml must be ignored, .streamlit/ directory must not**
Add `.streamlit/secrets.toml` to both `.gitignore` and `.gcloudignore` — it contains real secrets and should never be uploaded to Cloud Build or committed to git. But do NOT ignore the `.streamlit/` directory itself — it contains `config.toml` (theme, server settings) which your app needs at runtime. In production, secrets come from `--set-secrets` env vars, not from `secrets.toml`.

---

## What's Next

- Deploy your Streamlit app → `cloud-run-deployment.md`
- Auth + session bookmarks need Supabase → `supabase-integration.md`
- Agent called from Streamlit via wrapper → `fastapi-adk-gateway.md`
