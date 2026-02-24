# Decisions: ghl-email-html-agent-v1

Key technical choices made during this build, with context and rationale.

---

## Decision 1: Global Module State for Tools (`_active_html`)

**Choice:** Store the active HTML in a module-level global variable in `tools.py`, accessed via `set_active_html()` / `get_active_html()`.

**Alternatives considered:**
- Pass HTML through every tool function parameter (stateless tools)
- Store HTML in ADK session state
- Return modified HTML from each tool and thread it through agent

**Why this:**
- ADK agents call tools as functions — they don't automatically thread return values into subsequent calls
- Passing HTML as a parameter would require the agent to know the full HTML on every tool call and pass it explicitly — fragile, token-wasteful
- Session state in ADK would require serialization and adds complexity
- Module-level global is the simplest reliable bridge: agent.py loads state before run, tools mutate it, agent.py reads it after run

**Trade-off:** Not thread-safe. Fine for single-user Streamlit apps (one user per process). Would need refactoring for multi-user or API server deployments.

---

## Decision 2: AgentTool for Google Search (Not Direct Integration)

**Choice:** Wrap `search_agent` (which has `google_search`) as `AgentTool`, add the wrapper to `ghl_agent.tools`.

**Alternatives considered:**
- Add `google_search` directly to `ghl_agent.tools`
- Implement custom search via `googlesearch-python` library as a FunctionTool
- Skip search capability entirely

**Why this:**
ADK throws `400 INVALID_ARGUMENT: Multiple tools are supported only when they are all search tools` when mixing Grounding tools (google_search is a Grounding tool) with FunctionTools in the same agent.

The custom `googlesearch-python` approach was tried first but required an extra dependency and didn't use Google's official Grounding API.

`AgentTool` is the ADK-native solution: isolate Grounding tools in a specialist agent, wrap it for use in a parent agent.

**Lesson for manual:** This is the canonical solution to the Grounding + FunctionTool mixing problem. Document this as a known ADK limitation.

---

## Decision 3: InMemorySessionService (Not DatabaseSessionService)

**Choice:** Use `InMemorySessionService` at global scope.

**Alternatives considered:**
- `DatabaseSessionService(db_url="sqlite:///./agent_sessions.db")` — commented out in agent.py
- Redis-backed session service
- No persistence (recreate session each call)

**Why this:**
- The "amnesia bug" was that `InMemorySessionService` was being recreated inside the function on every call — the fix is to make it module-global, not to switch storage backends
- SQLite would accumulate growing session data with no cleanup strategy in place
- For a single-user Streamlit demo app, in-memory + global scope is sufficient
- Sessions only need to persist for the duration of a browser tab, not across server restarts

**When to upgrade:** If the app needs to persist agent history across server restarts, across users, or at scale → switch to `DatabaseSessionService` with SQLite for single-user or Supabase/Postgres for multi-user.

---

## Decision 4: Sync Wrapper Around Async ADK (`asyncio.run`)

**Choice:** `run_agent()` is synchronous; it calls `asyncio.run(_run_async(...))` internally.

**Alternatives considered:**
- Expose async interface to Streamlit and use `asyncio` in main.py
- Use a thread pool executor

**Why this:**
- Streamlit's execution model is synchronous — it re-runs the script top to bottom
- Streamlit has its own event loop management; nesting async inside Streamlit requires care
- Cleanest pattern: agent.py exposes a sync function, internally manages its own async context via `asyncio.run()`
- main.py doesn't need to know anything about async

**Trade-off:** Can't use `await` at the Streamlit level. Fine for this app since there's only one async operation per user action.

---

## Decision 5: HTML Context Injected in Every Prompt (Not in System Instruction)

**Choice:** Inject current HTML into the user prompt on every turn.

**Alternatives considered:**
- Inject HTML into agent's `instruction` field (system prompt)
- Store HTML in ADK session state and reference it
- Have agent maintain internal HTML state from tool return values

**Why this:**
- System instruction is set at agent initialization — it can't change to reflect the current HTML state
- Injecting into the user prompt means the agent always sees the latest version
- After each tool call, the HTML changes — the next turn injects the updated HTML
- Simple and explicit; no hidden state assumptions

**Trade-off:** Large HTML templates consume significant tokens on every turn. For very large templates (100KB+), this could be expensive. For typical GHL email templates (5-30KB), it's fine.

---

## Decision 6: BeautifulSoup for HTML Manipulation (Not Regex or LLM Output)

**Choice:** Tools use BeautifulSoup to surgically modify HTML; the agent never outputs raw HTML.

**Alternatives considered:**
- Agent generates the full modified HTML as text output → replace in UI
- Regex-based string replacement
- Direct DOM manipulation via JavaScript (Streamlit component)

**Why this:**
- LLM-generated HTML output is fragile for large templates — LLMs hallucinate, truncate, and introduce subtle changes
- Regex is brittle for nested HTML structures
- BeautifulSoup is deterministic: `soup.select(selector)` finds exactly the right elements, `.style` attribute dict manipulation is safe
- Agent acts as an **intent interpreter** (what to change), tools act as the **executor** (how to change it) — clean separation

**Result:** Agent reliability is dramatically higher. Even if the agent makes a wrong selector choice, it can be corrected. The HTML structure is never corrupted by LLM output.

---

## Decision 7: Selector-Based Tool API (CSS Selectors, Not Positional)

**Choice:** All tools accept CSS selectors (`selector: str`) to identify target elements.

**Alternatives considered:**
- XPath selectors
- Positional indices (e.g., "3rd button")
- Custom element IDs injected into HTML

**Why this:**
- CSS selectors are what LLMs know best — they're in every web dev training corpus
- `soup.select()` supports the full CSS selector spec
- Agents can use class names (`a.cta-button`), attribute selectors (`a[style*='red']`), or element types (`h1`)
- Natural language user requests map well to CSS selectors (LLM knows "the red button" → `a[style*='background.*red']`)

---

## Decision 8: `asyncio.run` Instead of `runner.run` (Sync ADK API)

**Choice:** Use `runner.run_async()` wrapped in `asyncio.run()` rather than any synchronous ADK runner API.

**Context:** ADK's `Runner` class has both `run()` (synchronous) and `run_async()` (async) APIs. The team chose `run_async()` wrapped manually.

**Why:** The async version gives access to streaming events (tool calls, response chunks) which enables the `status_callback` pattern. The sync `run()` returns only the final response with no intermediate events.

---

## Decision 9: Two-Screen UI Pattern (Load → Editor)

**Choice:** Single Streamlit file with conditional layout based on `st.session_state.current_html`.

**Alternatives considered:**
- Two separate pages (`pages/load.py`, `pages/editor.py`)
- Modal/dialog for HTML input
- File upload widget instead of text paste

**Why this:**
- Simple text paste is fastest for testing with GHL HTML (copy from GHL → paste → edit)
- Multi-page Streamlit adds complexity and requires `gatekeeper()` logic; unnecessary for single-user tool
- Conditional layout in one file is more maintainable for a focused-scope tool

---

## Decision 10: No Cloud Deployment (Local Tool)

**Choice:** No Dockerfile, no Cloud Run, no `deploy.sh`. Local Streamlit only.

**Context:** Previous repos in the series all target Cloud Run deployment. This one doesn't.

**Why this:**
- This is a personal/demo tool for editing GHL templates, not a production service
- HTML editing with an AI agent requires direct Vertex AI access — local credentials are sufficient
- No multi-user requirement; no need for containerization at this stage
- `.org` pattern still present in data/ (confirms the convention is established, not related to deployment)

**Implication for manuals:** Local-only apps with Vertex AI still follow the `GOOGLE_GENAI_USE_VERTEXAI=1` + project/location env var pattern. No `gcloud run deploy` needed for local use.
