# Key Decisions: ADK Wrapper (google-adk-wrapper-v2)

---

## Decision 1: Stateless Wrapper (No Database)

**Choice:** The wrapper holds zero state. No database, no in-memory session store.

**Rationale:** The ADK bundle already manages sessions (via Supabase in production). The wrapper's job is routing and adaptation, not state management. Adding state would duplicate the bundle's responsibility and break Cloud Run's scale-to-zero model.

**Alternative rejected:** Caching sessions or responses in-memory. This would break on restarts and create sync issues with the ADK bundle's actual state.

**Impact:** Any Cloud Run instance can handle any request. Horizontal scaling is trivial. Cold starts are unaffected by state.

---

## Decision 2: Single Base URL Per Environment (All Agents Share One URL)

**Choice:** `AGENT_REGISTRY = { agent_name: base_url }` where `base_url` is the same for all agents.

**Rationale:** The ADK bundle deploys all agents as a single Cloud Run service. Each agent is addressed via the `app_name` parameter in API calls (`/apps/{app_name}/...`), not via separate URLs.

**Alternative rejected:** Separate URL per agent. This would require a separate Cloud Run service per agent, which doesn't match how `adk api_server .` works (it discovers all agents in the project directory and serves them under one server).

**Implication:** If you ever split agents into multiple bundles, the config.json structure would need to change from `{environments.env.adk_bundle_url}` to a per-agent URL map.

---

## Decision 3: Session Creation in Wrapper, Not Frontend

**Choice:** Wrapper creates sessions when needed. Frontend only needs to pass `session_id` back on subsequent calls.

**Rationale:** ADK's session creation API (`POST /apps/{app}/users/{user}/sessions/{id}`) is implementation-specific. If the wrapper exposes this complexity to the frontend, every frontend has to implement it correctly. The wrapper handles it so the frontend just sends `{message, user_id, session_id?}`.

**Result:** Frontend session management reduces to: "save the `session_id` from the first response, send it back on all subsequent requests."

---

## Decision 4: 404 Auto-Recovery (Create New Session on Stale ID)

**Choice:** On ADK 404 response, create a new session and retry once. Return the new session_id in the response.

**Rationale:** Sessions can expire (ADK bundle restart, Supabase cleanup, or the session simply never existed). Without recovery, users would see hard errors on session expiry. The wrapper silently handles it and returns a fresh session_id.

**Trade-off:** Conversation history is lost when a new session is created. This is acceptable because the session was already gone — the alternative (returning an error) is worse for UX.

**Limitation:** Only retries once. If the new session creation also fails, the error propagates.

---

## Decision 5: config.json + config.py (Not Pure Env Vars, Not Hardcoded)

**Choice:** Agent list and URLs live in `config.json`. The env var `APP_ENV` selects the environment. `config.py` builds the registry at startup.

**Rationale:**
- Hardcoding agent names in Python: requires code change to add an agent
- Pure env vars: hard to pass a list of 5+ agents cleanly
- config.json: single file to update when adding agents or changing URLs, version-controlled, auditable

**Separation of concerns:**
- `config.json` = what agents exist and where the bundle lives (data)
- `APP_ENV` = which environment to use (runtime selection)
- `config.py` = how to build the registry (logic)

---

## Decision 6: APP_ENV Toggle for Local/Cloud Targeting

**Choice:** `APP_ENV=local` (default) → localhost. `APP_ENV=cloud` → live Cloud Run URL.

**Rationale:** Supports three development modes:
1. Full local (wrapper + bundle both local) — fast, no cloud costs
2. Hybrid (local wrapper → cloud bundle) — test new wrapper code against real agents
3. Full cloud (both deployed) — production

`start_cloud.sh` makes mode 2 a one-command operation.

**Note:** `Dockerfile` bakes `APP_ENV=cloud` in so the deployed wrapper always points to the live bundle, regardless of what's in `config.json`'s local section.

---

## Decision 7: History Endpoint in Wrapper, Not Direct ADK Access

**Choice:** Frontend calls `/get_history` on the wrapper, which fetches from ADK and normalizes the response.

**Rationale:** ADK returns raw events (complex JSON with author, content, parts, etc.). The normalized format (`[{role, content}]`) is what every chat UI needs. If frontend fetched ADK directly, every frontend would re-implement the same normalization logic.

**Result:** History is a single source of truth via the wrapper. Changing ADK's event format only requires updating `_normalize_events()` in one place.

---

## Decision 8: httpx (Async) Instead of requests (Sync)

**Choice:** `httpx.AsyncClient` for all outbound HTTP calls.

**Rationale:** FastAPI is built on async. Using synchronous `requests` library inside an async endpoint would block the event loop. `httpx` provides the same API as `requests` but is fully async-compatible.

**Timeout values:**
- Session creation: 10s (fast, just creates a record)
- Agent run: 60s (LLM calls can be slow)
- History fetch: 30s (reading, not writing or running LLM)

---

## Decision 9: No Authentication (--allow-unauthenticated)

**Choice:** Wrapper is publicly accessible with no API key or token.

**Rationale:** Consistent with the other repos in this system. All services in the current stack use `--allow-unauthenticated`. This is appropriate for demo/development but would need to change for production.

**What production auth would look like:**
- API key in `X-API-Key` header validated by FastAPI middleware
- Or Cloud Run IAM invoker role — caller needs a service account

---

## Decision 10: No Streaming

**Choice:** Wait for the full ADK response, then return everything at once.

**Rationale:** ADK has a `/run_sse` endpoint for streaming. This wrapper uses `/run` (non-streaming). The tradeoff is simplicity vs. latency — the user waits for the full response before seeing anything.

**When to change:** For conversational UIs where users expect streaming output (like ChatGPT-style typing effect), switch to `/run_sse` and use FastAPI's `StreamingResponse`.

---

## Decision 11: config-org.json and main-org.py as Backup Pattern

**Choice:** Keep `.org` versions of key files.

**Rationale:** Same pattern seen in `adk-exp-v2` (`.org.py` files). Developer preserves the known-working version before making changes. Low-tech but effective "undo" mechanism for exploratory edits.

**Pattern:** When about to modify a working file, copy it as `filename-org.ext` first.
