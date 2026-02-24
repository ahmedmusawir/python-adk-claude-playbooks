# Supabase Integration

## What This Is For

Add auth and session persistence to any Python/Streamlit app. Covers the Supabase email/password auth gate, multi-page protection, per-user ADK session bookmarks, and injection patterns for Cloud Run.

Sourced from ADK Streamlit V2 (production Streamlit + Supabase app serving a GHL CRM agent).

---

## Quick Reference — Auth Gate

```python
# app.py
import streamlit as st
from supabase import create_client, Client
import os

def get_supabase() -> Client:
    """Reads from st.secrets (local) or env vars (Cloud Run)."""
    try:
        url = st.secrets["SUPABASE_URL"]
        key = st.secrets["SUPABASE_ANON_KEY"]
    except (KeyError, FileNotFoundError):
        url = os.environ["SUPABASE_URL"]
        key = os.environ["SUPABASE_ANON_KEY"]
    return create_client(url, key)

def main():
    st.set_page_config(page_title="My App")

    if "supabase_session" not in st.session_state:
        st.session_state["supabase_session"] = None
        st.session_state["supabase_user"] = None

    if st.session_state["supabase_session"] is None:
        show_login()
        return

    show_main_app()

def show_login():
    st.title("Sign In")
    email = st.text_input("Email")
    password = st.text_input("Password", type="password")

    if st.button("Sign In"):
        try:
            result = get_supabase().auth.sign_in_with_password({
                "email": email, "password": password
            })
            st.session_state["supabase_session"] = result.session
            st.session_state["supabase_user"] = result.user
            st.rerun()
        except Exception as e:
            st.error(f"Login failed: {str(e)}")

if __name__ == "__main__":
    main()
```

---

## Pattern 1: Client Setup — st.secrets + Secret Manager Dual Path

Same code works locally (reads from `.streamlit/secrets.toml`) and in Cloud Run (reads from env vars injected from Secret Manager).

```python
import streamlit as st
import os
from supabase import create_client, Client

def get_supabase() -> Client:
    """Reads from st.secrets (local) or env vars (Cloud Run)."""
    try:
        url = st.secrets["SUPABASE_URL"]
        key = st.secrets["SUPABASE_ANON_KEY"]
    except (KeyError, FileNotFoundError):
        url = os.environ["SUPABASE_URL"]
        key = os.environ["SUPABASE_ANON_KEY"]
    return create_client(url, key)
```

`.streamlit/secrets.toml` (local, gitignored):
```toml
SUPABASE_URL = "https://xxxxx.supabase.co"
SUPABASE_ANON_KEY = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

Cloud Run deploy:
```bash
gcloud run deploy my-app \
  --set-secrets="SUPABASE_URL=supabase-url:latest,SUPABASE_ANON_KEY=supabase-anon-key:latest"
```

---

## Pattern 2: Full Auth Gate with Error Handling

```python
def show_login():
    st.title("Sign In")

    with st.form("login_form"):
        email = st.text_input("Email")
        password = st.text_input("Password", type="password")
        submitted = st.form_submit_button("Sign In")

    if submitted:
        if not email or not password:
            st.error("Email and password are required.")
            return

        try:
            supabase = get_supabase()
            result = supabase.auth.sign_in_with_password({
                "email": email,
                "password": password,
            })

            if result.session is None:
                st.error("Authentication failed.")
                return

            st.session_state["supabase_session"] = result.session
            st.session_state["supabase_user"] = result.user
            st.session_state["user_id"] = result.user.id
            st.rerun()

        except Exception as e:
            error_msg = str(e).lower()
            if "invalid" in error_msg or "credentials" in error_msg:
                st.error("Invalid email or password.")
            else:
                st.error(f"Login error: {e}")
```

---

## Pattern 3: gatekeeper() — Multi-Page Protection

For multi-page apps (`pages/` directory). Call at the top of every protected page. One call = page stops rendering if not authenticated.

```python
# utils/auth.py
import streamlit as st

def gatekeeper():
    """
    Call at the top of any page requiring authentication.
    Stops page rendering with a warning if not authenticated.
    """
    if st.session_state.get("supabase_session") is None:
        st.warning("Please sign in to access this page.")
        st.stop()
```

```python
# pages/02_Chat.py
from utils.auth import gatekeeper
gatekeeper()  # ← nothing below this renders if not authenticated

st.title("Chat")
st.write("Welcome back! Start chatting below.")
```

---

## Pattern 4: Per-User Session Bookmarks

Each user has a `user_profiles` row in Supabase. The `session_bookmarks` column holds a JSON blob: `{agent_name: session_id}`. When users return, they resume the same ADK session.

### Supabase Table Setup

```sql
-- Run in Supabase SQL editor
CREATE TABLE user_profiles (
    user_id UUID PRIMARY KEY REFERENCES auth.users(id),
    session_bookmarks JSONB DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Row level security: users can only see their own profile
ALTER TABLE user_profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can read own profile"
    ON user_profiles FOR SELECT
    USING (auth.uid() = user_id);

CREATE POLICY "Users can update own profile"
    ON user_profiles FOR ALL
    USING (auth.uid() = user_id);
```

### Python: Load and Save Bookmarks

```python
import json
from supabase import Client

def load_session_bookmarks(supabase: Client, user_id: str) -> dict:
    """Load {agent_name: session_id} map for this user."""
    try:
        result = (
            supabase.table("user_profiles")
            .select("session_bookmarks")
            .eq("user_id", user_id)
            .execute()
        )
        if result.data:
            bookmarks = result.data[0].get("session_bookmarks", {})
            # Handle both string and dict (Supabase JSONB can return either)
            if isinstance(bookmarks, str):
                return json.loads(bookmarks)
            return bookmarks or {}
    except Exception as e:
        print(f"Failed to load bookmarks: {e}")
    return {}


def save_session_bookmark(
    supabase: Client, user_id: str, agent_name: str, session_id: str
) -> None:
    """Upsert session_id for this user+agent."""
    try:
        bookmarks = load_session_bookmarks(supabase, user_id)
        bookmarks[agent_name] = session_id

        from datetime import datetime, timezone
        supabase.table("user_profiles").upsert({
            "user_id": user_id,
            "session_bookmarks": bookmarks,
            "updated_at": datetime.now(timezone.utc).isoformat(),
        }).execute()
    except Exception as e:
        print(f"Failed to save bookmark: {e}")  # Non-fatal — don't crash the app
```

### Session Bookmark Flow

```python
import uuid

def get_or_create_agent_session(
    supabase: Client, user_id: str, agent_name: str
) -> str:
    """Return existing session_id or create new one."""
    bookmarks = load_session_bookmarks(supabase, user_id)

    if agent_name in bookmarks and bookmarks[agent_name]:
        return bookmarks[agent_name]

    # New session
    session_id = f"{agent_name}-{user_id[:8]}-{uuid.uuid4().hex[:8]}"
    save_session_bookmark(supabase, user_id, agent_name, session_id)
    return session_id


# At app startup (after auth):
if st.session_state.get("agent_session_id") is None:
    supabase = get_supabase()
    user_id = st.session_state["supabase_user"].id
    agent_name = st.session_state.get("selected_agent", "default_agent")

    st.session_state["agent_session_id"] = get_or_create_agent_session(
        supabase, user_id, agent_name
    )
```

---

## Pattern 5: ADK Session Persistence Across Logins

The bookmark system (above) stores session IDs. On next login, the ADK session history is still in the ADK backend's session store. This only works if:

1. The ADK backend uses a persistent session store (not `InMemorySessionService`)
2. The session ID is consistent across logins

For Cloud Run ADK agents, use `--session_service_uri` with a Postgres/Supabase connection:

```bash
# In deploy.sh for your ADK backend:
gcloud run deploy adk-bundle \
  --set-secrets="DB_URI=session-db-uri:latest"
```

```dockerfile
# In Dockerfile:
CMD adk api_server . --port $PORT --session_service_uri=$DB_URI
```

The `DB_URI` is your Supabase Postgres connection string: `postgresql://postgres:[password]@db.[project].supabase.co:5432/postgres`

---

## Pattern 6: Logout

```python
def logout():
    """Sign out from Supabase and clear all session state."""
    try:
        get_supabase().auth.sign_out()
    except Exception:
        pass  # Still clear local state even if Supabase sign-out fails

    # Clear all Streamlit session state
    for key in list(st.session_state.keys()):
        del st.session_state[key]

    st.rerun()

# In sidebar:
if st.sidebar.button("Sign Out"):
    logout()
```

---

## Pattern 7: User Profile Data

Store any per-user config in a JSONB `preferences` column on `user_profiles`. Add it to the table:

```sql
-- Add preferences column (run once in Supabase SQL editor)
ALTER TABLE user_profiles ADD COLUMN IF NOT EXISTS preferences JSONB DEFAULT '{}'::jsonb;
```

```python
def save_user_preference(supabase: Client, user_id: str, key: str, value) -> None:
    """Save a preference to user profile's JSONB preferences column."""
    prefs = load_all_preferences(supabase, user_id)
    prefs[key] = value
    supabase.table("user_profiles").upsert({
        "user_id": user_id,
        "preferences": prefs,
    }).execute()

def load_all_preferences(supabase: Client, user_id: str) -> dict:
    """Load all preferences for a user."""
    result = (
        supabase.table("user_profiles")
        .select("preferences")
        .eq("user_id", user_id)
        .execute()
    )
    if result.data:
        return result.data[0].get("preferences", {}) or {}
    return {}

def load_user_preference(supabase: Client, user_id: str, key: str, default=None):
    """Load a single preference by key."""
    prefs = load_all_preferences(supabase, user_id)
    return prefs.get(key, default)

# Example: save preferred agent
save_user_preference(supabase, user_id, "preferred_agent", "email_agent")
agent = load_user_preference(supabase, user_id, "preferred_agent", "default_agent")
```

---

## Supabase in Cloud Run

### Inject Credentials from Secret Manager

```bash
gcloud run deploy my-streamlit-app \
  --set-secrets="SUPABASE_URL=supabase-url:latest,SUPABASE_ANON_KEY=supabase-anon-key:latest"
```

For server-side operations (RLS bypass), also inject the service role key:
```bash
gcloud run deploy my-streamlit-app \
  --set-secrets="SUPABASE_URL=supabase-url:latest,SUPABASE_ANON_KEY=supabase-anon-key:latest,SUPABASE_SERVICE_ROLE_KEY=supabase-service-role-key:latest"
```

**Never commit the service role key to source control.** Use Secret Manager.

### Connection Pooling for High Traffic

Supabase's default Postgres connection limit is 100 (free tier) to 500+ (paid). If your Cloud Run service scales to many instances, each instance opens its own connection pool.

For high-traffic apps, use Supabase's connection pooler (Transaction pooler):
```
# Regular connection (for long-lived connections):
postgresql://postgres:[password]@db.[project].supabase.co:5432/postgres

# Transaction pooler (for serverless/many short connections):
postgresql://postgres.[project]:[password]@aws-0-us-east-1.pooler.supabase.com:6543/postgres
```

Use the transaction pooler for Cloud Run `--session_service_uri`.

---

## Row Level Security Quick Setup

For any table that stores user data, enable RLS:

```sql
-- Enable RLS
ALTER TABLE your_table ENABLE ROW LEVEL SECURITY;

-- Users can only CRUD their own rows
CREATE POLICY "user_own_data" ON your_table
    USING (auth.uid() = user_id)
    WITH CHECK (auth.uid() = user_id);
```

With RLS enabled, the `anon` key can only see rows where `user_id = auth.uid()`. No additional filtering needed in Python code.

---

## requirements.txt

```
supabase==2.3.0
streamlit>=1.30.0
```

**Use supabase v2.x.** The API changed significantly from v1. If you see `supabase.table()` not working, check your version.

---

## Gotchas

**1. result.session can be None on success**
Some auth flows (magic link, email confirmation) return a session as `None` even on success. Always check `if result.session is None` before storing it.

**2. JSONB returns dict, not string**
Supabase's Python client returns JSONB columns as Python dicts. But if you stored the value as a JSON string (e.g., `json.dumps(bookmarks)`), you get back a string. Be consistent — let Supabase handle JSON natively by passing dicts directly.

**3. upsert requires the primary key**
`supabase.table("user_profiles").upsert({...})` requires `user_id` to be in the dict for the upsert to work. Without it, Supabase doesn't know which row to update.

**4. RLS blocks server-side operations**
If your Cloud Run backend needs to read any user's data (not just the logged-in user's), use the service role key, not the anon key. The service role key bypasses RLS.

**5. Supabase anon key is public**
The anon key is meant to be used in the browser/client. It's safe to include in Streamlit apps (accessible to users). The service role key is private and must only be used server-side.

**6. st.session_state["supabase_session"] holds a session object**
The session object has a `access_token` that expires. For long-running apps, you may need to refresh the session. `supabase.auth.refresh_session()` handles this.

---

## What's Next

- Build the Streamlit UI using this auth system → `streamlit-patterns.md`
- Deploy to Cloud Run with secrets → `cloud-run-deployment.md`
- Supabase as ADK session backend → `fastapi-adk-gateway.md`
