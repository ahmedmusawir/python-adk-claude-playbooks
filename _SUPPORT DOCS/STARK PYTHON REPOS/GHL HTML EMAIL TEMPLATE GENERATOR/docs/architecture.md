# Architecture: ghl-email-html-agent-v1

**App Type:** Streamlit + ADK + BeautifulSoup HTML Editor
**Purpose:** AI-powered GHL email template editor — chat with an agent to modify HTML

---

## What This App Does

User pastes a GHL email HTML template. An ADK agent (backed by Gemini 2.5 Flash via Vertex AI) receives natural language instructions ("make the button blue", "change the headline text") and calls BeautifulSoup-powered tools to surgically modify the HTML. Live preview updates after each change. Undo stack lets user revert changes.

---

## File Structure

```
ghl-email-html-agent-v1/
├── main.py              # Streamlit UI (all UI logic lives here)
├── agent.py             # ADK agent definition + async runner + session management
├── tools.py             # BeautifulSoup HTML manipulation tools + global state
├── requirements.txt     # streamlit, google-adk, beautifulsoup4, python-dotenv, google-genai
├── data/
│   ├── sample_ghl.html       # Sample GHL email template for testing
│   └── sample_ghl-org.html   # Original backup (.org pattern)
├── output.html          # Tool test harness output
├── test_payload.json    # JSON payload for testing tools via CLI
├── inspect_code/        # Inspection/debugging scripts ONLY (never in root)
│   ├── check_imports.py
│   ├── check_adk_imports.py
│   ├── inspect_agent.py
│   └── ...
├── progress_info/       # Session logs documenting build history
│   ├── session_1.md
│   └── session_2.md
└── .cursorrules         # Rule: UI in main.py, agent in agent.py, utils in inspect_code/
```

---

## System Flow

```
User (browser)
    ↓ paste HTML → click "Start Editing"
main.py (Streamlit)
    ↓ st.session_state.current_html populated
    ↓ user types chat message
    ↓ save_state() → push to html_history stack
    ↓ agent.run_agent(user_input, current_html, session_id, status_callback)
        ↓ tools.set_active_html(current_html)          # load HTML into global state
        ↓ asyncio.run(_run_async(...))
            ↓ get_or_create ADK session
            ↓ Runner.run_async(prompt=f"HTML:\n{html}\n\nUSER: {user_input}")
            ↓ ADK agent → Gemini 2.5 Flash (Vertex AI)
            ↓ agent calls tools (apply_style_edit, update_text_content, etc.)
                ↓ each tool: read _active_html → BeautifulSoup parse → modify → write _active_html
            ↓ on function_call event → status_callback(tool_name) → st.toast in UI
        ↓ tools.get_active_html()                      # retrieve modified HTML
    ↓ return (response_text, new_html)
    ↓ st.session_state.current_html = new_html
    ↓ st.rerun()
User sees updated preview
```

---

## Three-File Architecture

### `tools.py` — Stateful Tool Layer

- Holds `_active_html` as a module-level global variable
- `set_active_html()` / `get_active_html()` are the state interface
- Each tool: accepts `selector` + params, reads `_active_html`, parses with BeautifulSoup, modifies, writes back to `_active_html`, returns modified HTML string
- Optional `html_content` parameter on every tool — allows direct invocation with HTML (used by CLI test harness)
- `__main__` block: JSON test harness for testing tools without ADK

### `agent.py` — ADK Agent + Runner

- Defines two agents: `search_agent` (google_search only) + `ghl_agent` (HTML tools + search_tool)
- `search_agent` wrapped as `AgentTool` and added to `ghl_agent.tools` — solves multi-tool type conflict
- `session_service = InMemorySessionService()` at **module scope** — persists across function calls
- `run_agent()` is the synchronous entry point; internally calls `asyncio.run(_run_async(...))`
- `_run_async()` handles: get-or-create session, build prompt with HTML context, run events loop, collect response

### `main.py` — Streamlit UI

- Two-screen pattern: load screen (if no HTML) → editor screen (if HTML loaded)
- Session state: `agent_session_id`, `html_history` (undo stack), `current_html`, `chat_history`
- Sidebar: Undo button + chat history + chat input
- Main area: tabs (Preview via `components.html()` + Code via `st.code()`)
- Status feedback: `st.toast()` triggered via callback on each tool call

---

## HTML State Flow Detail

The key bridge between stateless LLM and stateful HTML:

```
main.py holds:  st.session_state.current_html
agent.py calls: tools.set_active_html(current_html)   ← loads before agent run
tools.py holds: _active_html (module global)           ← mutated by each tool
agent.py reads: tools.get_active_html()                ← retrieves after agent run
main.py stores: st.session_state.current_html = new_html
```

Every agent run: load → mutate → retrieve. Never relies on agent to return HTML.

---

## ADK Session Pattern

```
InMemorySessionService (module-level global)
    ↓
get_session() → returns None if missing (not exception)
    ↓ if None:
create_session()
    ↓
Runner(agent, app_name, session_service)
    ↓
run_async(user_id, session_id, new_message)
```

Session ID generated once in `st.session_state.agent_session_id` → passed on every `run_agent()` call → agent remembers conversation history within a Streamlit session.

---

## Tool Suite

| Tool | Purpose | Key Behavior |
|------|---------|--------------|
| `apply_style_edit` | CSS property change | Parses inline style → dict → update → reconstruct |
| `update_text_content` | Text replacement | `element.string = new_text` |
| `update_inner_html` | Rich HTML replacement | `element.clear()` + parse + append fragments |
| `insert_element_relative` | Add HTML elements | `before`, `after`, `inside_start`, `inside_end` |
| `remove_element` | Delete elements | `element.decompose()` |

All tools accept CSS selectors (via `soup.select()`). Error strings prefixed with `"ERROR:"` returned on failure.

---

## Key Dependencies

| Package | Role |
|---------|------|
| `streamlit` | UI framework |
| `google-adk` | Agent runtime (Runner, InMemorySessionService, Agent, AgentTool) |
| `beautifulsoup4` | HTML parsing + surgical DOM manipulation |
| `google-genai` | Vertex AI SDK for Gemini |
| `python-dotenv` | Local env var loading |

---

## Environment Setup

```bash
# .env or environment
GOOGLE_CLOUD_PROJECT=your-project-id
GOOGLE_CLOUD_LOCATION=us-central1

# agent.py sets this at module level:
os.environ["GOOGLE_GENAI_USE_VERTEXAI"] = "1"
```

No explicit `genai.Client()` construction needed — `GOOGLE_GENAI_USE_VERTEXAI=1` switches the SDK to Vertex AI automatically.
