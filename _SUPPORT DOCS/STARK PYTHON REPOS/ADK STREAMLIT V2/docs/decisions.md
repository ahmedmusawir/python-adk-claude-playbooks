# Streamlit Frontend Key Decisions

**Project:** google-adk-n8n-hybrid-streamlit-v2
**Extracted:** 2026-02-23

---

## Decision 1: Streamlit as Frontend (Not React/Next.js)

**Context:**
Need a user-facing chat interface for the ADK agent system.

**Decision:**
Use Streamlit.

**Alternatives Considered:**
1. **React + Next.js** — Full web app stack
2. **Streamlit** — Python-native rapid UI

**Rationale:**
- Team is Python-focused; no frontend expertise needed
- Streamlit chat primitives (`st.chat_message`, `st.chat_input`) are production-ready
- Same codebase can be used for admin tools (Mission Control)
- Deploy to Cloud Run same as any other service (no separate hosting)
- Development speed: operational in hours not days

**Trade-offs:**
- ✅ Python-native (no JS context switch)
- ✅ Built-in chat widgets
- ✅ Fast to build and iterate
- ❌ Limited UI customization vs React
- ❌ Not suitable for complex interactive UIs (forms, dashboards)
- ❌ Streamlit's execution model (full script rerun on every interaction) requires careful state management

**Outcome:**
Correct choice for an internal chat UI targeting the team. Would revisit for public-facing product.

---

## Decision 2: Supabase for Both Authentication and Session Bookmarks

**Context:**
Need user authentication AND a place to store which ADK session each user has per agent.

**Decision:**
Use Supabase for both (single service, two functions).

**Alternatives Considered:**
1. **Supabase auth only** — Store sessions in browser cookies or local storage
2. **Supabase for both** — Auth + custom profiles table
3. **Firebase Auth + Firestore** — Google-native alternative
4. **Custom auth + Postgres** — Full control, more complexity

**Rationale:**
- Supabase is already the session database for the ADK bundle (Repo #4)
- One service account, one connection setup, one billing item
- The `adk_n8n_hybrid_profiles` table is trivially simple (id + JSON blob)
- `upsert` handles both new and returning users with one operation
- Sessions persist across logins and devices (vs browser storage which is device-local)

**Trade-offs:**
- ✅ Single service for auth + data
- ✅ Cloud-native persistence (survives browser clearing)
- ✅ Multi-device support (same sessions on phone + laptop)
- ❌ Round-trip to Supabase on every agent switch
- ❌ Data stored as JSON blob (no relational queries on sessions)

**Outcome:**
Elegant solution. The `agent_sessions` JSON blob approach is appropriate for this scale.

---

## Decision 3: Proxy Through ADK Wrapper (Not Direct to N8N or ADK)

**Context:**
V1 (`chat-org.py`) had the frontend talking to N8N for messages AND directly to ADK for history. This required knowing 3 different service protocols.

**Decision:**
All communication goes through the ADK Wrapper only.

**V1 architecture:**
```
Frontend → N8N webhook (chat)
Frontend → ADK server /apps/{agent}/users/{uid}/sessions/{sid} (history)
```

**V2 architecture (current):**
```
Frontend → ADK Wrapper /run_agent (chat)
Frontend → ADK Wrapper /get_history (history)
```

**Rationale:**
- Frontend only needs one URL (WRAPPER_URL)
- Simplified response format (clean JSON vs N8N's double-JSON wrapping)
- Session 404 recovery handled by wrapper (not frontend)
- Backend services (N8N, ADK) can change without frontend changes
- ADK session URL structure (`/apps/{agent}/users/{uid}/sessions/{sid}`) is complex — wrapper abstracts it

**Trade-offs:**
- ✅ Frontend is simpler (1 URL, 2 endpoints)
- ✅ Decoupled from backend protocol changes
- ✅ Error recovery in wrapper not frontend
- ❌ Additional network hop (frontend → wrapper → ADK)
- ❌ Wrapper is a single point of failure (but it's simple/stateless)

**Outcome:**
The correct architecture. The wrapper's purpose is precisely this simplification.

---

## Decision 4: Procfile Instead of Dockerfile

**Context:**
Need to specify how Cloud Run starts the Streamlit application.

**Decision:**
Use `Procfile` instead of `Dockerfile`.

```
# Procfile
web: streamlit run chat.py --server.port $PORT --server.enableCORS false
```

**Alternatives Considered:**
1. **Dockerfile** — Used by all other services in the ecosystem (ADK Bundle, Wrapper)
2. **Procfile** — Simple text file, picked up by Cloud Run buildpack

**Rationale:**
- Streamlit apps have no custom build steps — no reason for a full Dockerfile
- Buildpack auto-detects Python + Streamlit, installs from requirements.txt
- `Procfile` is one line vs 10+ lines of Dockerfile
- `$PORT` substitution works the same way (shell-form)
- Less to maintain

**Trade-offs:**
- ✅ Much simpler than Dockerfile
- ✅ Buildpack handles Python version selection
- ❌ Less control over build environment
- ❌ Slightly slower builds (buildpack does more detection work)

**Pattern note:** For pure Python apps with no custom build, `Procfile` is the minimal-complexity choice. For ADK/uvicorn apps (non-Streamlit), `Dockerfile` with shell-form CMD is still required.

---

## Decision 5: `st.secrets` for Credentials (Not `os.getenv`)

**Context:**
Streamlit apps need to access secrets (Supabase credentials).

**Decision:**
Use `st.secrets["KEY"]` instead of `os.getenv("KEY")`.

**Alternatives Considered:**
1. **`os.getenv()`** — Standard Python
2. **`st.secrets["KEY"]`** — Streamlit-native

**Rationale:**
- `st.secrets` reads from `.streamlit/secrets.toml` locally
- Same key works when Secret Manager injects values via `--set-secrets` in deploy.sh
- No code change needed between local and cloud
- Streamlit documentation recommends this pattern

**How it works in Cloud Run:**
```bash
# deploy.sh:
--set-secrets="SUPABASE_URL=supabase-url:latest,SUPABASE_KEY=supabase-key:latest"
```
Cloud Run injects these as environment variables. Streamlit reads env vars into `st.secrets` automatically.

**Trade-offs:**
- ✅ Single code path for local + cloud
- ✅ Native Streamlit pattern
- ❌ Streamlit-specific (not portable to FastAPI, etc.)

---

## Decision 6: Single Table for User Profiles (`upsert` Pattern)

**Context:**
Need to store per-user, per-agent session IDs.

**Decision:**
One table (`adk_n8n_hybrid_profiles`), one column (`agent_sessions` as JSONB), `upsert` for all writes.

**Alternatives Considered:**
1. **Two tables** — `users` + `user_agent_sessions` (normalized relational)
2. **One table with JSONB blob** — Simple, flexible
3. **Redis** — Fast key-value store
4. **Browser storage** — No server needed

**Rationale:**
- The data is simple: a dict of `{agent_name: session_id}` per user
- JSONB in Postgres is efficient for small JSON documents
- `upsert` eliminates the "check if exists + insert or update" pattern
- Using Supabase user UUID as primary key links to auth without foreign key complexity
- Can add more profile fields later without schema changes

**Schema:**
```sql
CREATE TABLE adk_n8n_hybrid_profiles (
    id UUID PRIMARY KEY,  -- = Supabase auth.users.id
    agent_sessions JSONB DEFAULT '{}'
);
```

**Trade-offs:**
- ✅ No schema migrations when adding new agents
- ✅ Simple read/write (one `select`, one `upsert`)
- ❌ No row-level history (can't see which sessions a user had in the past)
- ❌ Overwrites on each save (last write wins)

---

## Decision 7: `gatekeeper()` Function for Multi-Page Auth

**Context:**
Streamlit multi-page apps (files in `pages/`) don't automatically share auth state — each page file runs independently.

**Decision:**
A single `gatekeeper()` function in `utils/auth.py` that any page calls at its top.

**Alternatives Considered:**
1. **Repeat login check in every page** — Duplicate code
2. **Centralized `gatekeeper()`** — DRY, reusable
3. **Streamlit custom component** — Overkill
4. **Session state check only** — No user feedback

**Rationale:**
- `st.stop()` cleanly halts execution without showing an error
- One function = consistent behavior across all pages
- Shows useful guidance (which page to log in on)
- Adding a new page requires one line, not copy-pasting logic

**Trade-offs:**
- ✅ DRY
- ✅ Consistent user experience across pages
- ✅ Easy to update auth behavior in one place
- ❌ Developer must remember to call it (not automatic)

---

## Decision 8: GCS for Agent Instructions (Not Database)

**Context:**
Mission Control page needs to edit agent instructions that the ADK bundle reads.

**Decision:**
Read from and write to the same GCS bucket that the ADK bundle uses for instructions.

**Rationale:**
- The ADK bundle already reads instructions from GCS via callable instruction pattern
- Writing to GCS = changes are live on next agent call (no redeploy)
- No intermediate API needed — `storage.Client()` works directly
- Consistent with the broader architecture (all agents read from GCS)

**ADC in Cloud Run:**
`storage.Client()` uses Application Default Credentials. In Cloud Run, the service account (`stark-vertex-ai`) has `roles/storage.objectAdmin` (set by `grant_streamlit_permissions.sh`). No key file needed.

---

## Decision 9: `chat-org.py` Preservation

**Context:**
Upgrading from V1 (N8N direct) to V2 (wrapper-only).

**Decision:**
Keep the V1 as `chat-org.py` before overwriting `chat.py`.

**Rationale:**
- Consistent with the `.org` file preservation pattern seen across the codebase
- Documents the evolution (what changed and why)
- Easy rollback if V2 has issues
- No cost to keeping it (it's not deployed)

---

## Key Decision Themes

**Simplicity over completeness:**
- `Procfile` over `Dockerfile`
- JSONB blob over normalized tables
- One function (`gatekeeper`) for auth

**Proxy pattern pays off:**
- Decision 3 (wrapper vs direct) is the most impactful architectural choice
- The frontend got dramatically simpler when V1 → V2

**Single service for multiple functions:**
- Supabase handles both auth and profile storage
- Streamlit handles both chat and admin (Mission Control)

---

_Extracted: 2026-02-23_
