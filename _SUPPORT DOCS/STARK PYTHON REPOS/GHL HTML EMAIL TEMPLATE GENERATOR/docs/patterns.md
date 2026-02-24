# Patterns: ghl-email-html-agent-v1

Copy-pasteable patterns extracted from this repo.

---

## Pattern 1: Global In-Module Tool State

Bridge between stateless LLM calls and stateful in-memory data. Used when tools need to operate on a shared object without passing it back and forth through the agent.

```python
# tools.py

_active_html = ""   # module-level global

def set_active_html(html_str: str):
    global _active_html
    _active_html = html_str

def get_active_html() -> str:
    return _active_html

def apply_style_edit(selector: str, css_property: str, css_value: str) -> str:
    global _active_html
    # ... modify _active_html ...
    return _active_html
```

```python
# agent.py — wrap the load/retrieve around agent execution
def run_agent(user_input: str, current_html: str, session_id: str = None):
    tools.set_active_html(current_html)          # 1. load
    asyncio.run(_run_async(user_input, ...))     # 2. agent runs tools (mutates state)
    new_html = tools.get_active_html()           # 3. retrieve
    return response_text, new_html
```

**When to use:** Any tool set that operates on a single document/object (HTML, JSON, XML). The agent doesn't need to handle the document — it just calls tools, the tools handle state internally.

---

## Pattern 2: AgentTool — Agent-as-a-Tool (Multi-Tool Conflict Fix)

ADK limitation: can't mix Grounding tools (google_search) with FunctionTools in the same agent. Solution: isolate the Grounding tool in a sub-agent, wrap it as `AgentTool`, add the wrapper to the parent.

```python
from google.adk.agents import Agent
from google.adk.tools import google_search
from google.adk.tools.agent_tool import AgentTool

# Step 1: Define specialist agent with ONLY grounding tools
search_agent = Agent(
    model="gemini-2.5-flash",
    name="search_specialist",
    description="A specialist in Google Search.",
    instruction="Use google_search to find factual information.",
    tools=[google_search]
)

# Step 2: Wrap it as a tool
search_tool = AgentTool(search_agent)

# Step 3: Add the wrapper (not the specialist agent) to the parent
main_agent = Agent(
    name="main_agent",
    model="gemini-2.5-flash",
    tools=[
        my_function_tool_1,
        my_function_tool_2,
        search_tool,       # ← AgentTool wraps the specialist
    ]
)
```

**When to use:** Any time you need to combine google_search (or other Grounding tools) with custom FunctionTools. The `AgentTool` wrapper is the only way to mix tool types in ADK.

---

## Pattern 3: ADK Session — Global Service + get_session() None Check

Session service must be module-level (not function-level) to survive across calls. `get_session()` returns `None` when the session doesn't exist — it does NOT raise an exception.

```python
# agent.py — module-level (not inside a function)
from google.adk.sessions import InMemorySessionService

session_service = InMemorySessionService()

async def _run_async(user_input: str, session_id: str):
    APP_NAME = "my_app"
    USER_ID = "user"

    # get_session returns None if session doesn't exist
    session = await session_service.get_session(
        app_name=APP_NAME,
        user_id=USER_ID,
        session_id=session_id
    )

    if session is None:   # ← explicit None check required
        session = await session_service.create_session(
            app_name=APP_NAME,
            user_id=USER_ID,
            session_id=session_id
        )

    runner = Runner(
        agent=my_agent,
        app_name=APP_NAME,
        session_service=session_service
    )
    # ...
```

**Gotcha:** If you recreate `InMemorySessionService()` inside the function, the agent loses history on every call ("amnesia bug"). Global scope = persistent within process lifetime.

---

## Pattern 4: status_callback for Tool Execution Feedback

Pass a callback from UI into the agent runner. Called on each `function_call` event. Gives the user real-time visibility into what the agent is doing.

```python
# agent.py
async def _run_async(user_input: str, ..., status_callback=None):
    events = runner.run_async(...)
    async for event in events:
        if hasattr(event, 'content') and event.content and event.content.parts:
            for part in event.content.parts:
                if part.text:
                    response_text += part.text
                if part.function_call:
                    if status_callback:
                        status_callback(part.function_call.name)  # ← fire callback
    return response_text

def run_agent(user_input: str, ..., status_callback=None):
    asyncio.run(_run_async(user_input, ..., status_callback=status_callback))
```

```python
# main.py — pass a lambda as the callback
response_text, new_html = agent.run_agent(
    user_input=prompt,
    current_html=st.session_state.current_html,
    session_id=st.session_state.agent_session_id,
    status_callback=lambda tool_name: st.toast(f"Agent is using tool: {tool_name}")
)
```

**When to use:** Any Streamlit app where you want the user to see live tool execution progress without blocking the UI.

---

## Pattern 5: HTML Context Injection in Prompt

Agent doesn't manage HTML state — it receives the full current HTML on every call. Simple and reliable for document editing workloads.

```python
# agent.py
full_prompt = f"""
CONTEXT:
You are editing the following HTML file.
```html
{html_context}
```

USER REQUEST:
{user_input}
"""

events = runner.run_async(
    user_id=USER_ID,
    session_id=session_id,
    new_message=types.Content(role="user", parts=[types.Part.from_text(text=full_prompt)])
)
```

**Why it works:** The agent sees the current document state on every turn. Tools mutate the global state. The agent doesn't need to "remember" the HTML — it gets injected fresh every time.

---

## Pattern 6: Two-Screen Streamlit App (Load → Editor)

Single-file Streamlit app with two modes. No routing library needed.

```python
# main.py
if not st.session_state.current_html:
    # --- Load Screen ---
    st.title("My HTML Editor")
    html_input = st.text_area("Paste HTML", height=300)
    if st.button("Start Editing"):
        st.session_state.current_html = html_input
        st.rerun()
else:
    # --- Editor Screen ---
    with st.sidebar:
        # ... chat controls ...
        pass

    tab1, tab2 = st.tabs(["Preview", "Code"])
    with tab1:
        components.html(st.session_state.current_html, height=800, scrolling=True)
    with tab2:
        st.code(st.session_state.current_html, language='html')
```

**When to use:** Any Streamlit app with a setup phase (upload, paste, configure) before the main interaction.

---

## Pattern 7: Undo Stack

Simple undo for any state value using a list as a stack.

```python
# main.py

# Initialize
if "html_history" not in st.session_state:
    st.session_state.html_history = []

def save_state():
    """Call BEFORE any mutation."""
    if st.session_state.current_html:
        st.session_state.html_history.append(st.session_state.current_html)

def undo():
    if st.session_state.html_history:
        st.session_state.current_html = st.session_state.html_history.pop()
        st.rerun()

# In UI
if st.button("Undo Last Change", disabled=len(st.session_state.html_history) == 0):
    undo()

# Before agent call
save_state()
response_text, new_html = agent.run_agent(...)
st.session_state.current_html = new_html
```

**When to use:** Any Streamlit editor that mutates state (HTML editing, image editing, config editing). Always call `save_state()` before the mutation.

---

## Pattern 8: BeautifulSoup CSS Surgery

Safely update a single CSS property in an inline style attribute without clobbering other properties.

```python
from bs4 import BeautifulSoup

def apply_style_edit(selector: str, css_property: str, css_value: str, html_content: str) -> str:
    soup = BeautifulSoup(html_content, 'html.parser')
    elements = soup.select(selector)

    for element in elements:
        current_style = element.get('style', '')
        style_dict = {}

        # Parse existing styles into dict
        if current_style:
            for style_pair in [s.strip() for s in current_style.split(';') if s.strip()]:
                if ':' in style_pair:
                    key, val = style_pair.split(':', 1)
                    style_dict[key.strip().lower()] = val.strip()

        # Update target property (add or overwrite)
        style_dict[css_property.strip().lower()] = css_value.strip()

        # Reconstruct style string
        new_style = '; '.join([f"{k}: {v}" for k, v in style_dict.items()]) + ';'
        element['style'] = new_style

    return str(soup)
```

**Why this over simple string replace:** Inline styles in HTML emails often have many properties. Simple `replace()` can clobber adjacent properties. Dict-based approach is surgical.

---

## Pattern 9: JSON Test Harness for Tool Functions

Test individual tool functions via CLI with a JSON payload — no agent stack, no Streamlit needed.

```json
// test_payload.json
{
  "function_name": "apply_style_edit",
  "args": {
    "selector": "h1",
    "css_property": "color",
    "css_value": "#FF0000"
  }
}
```

```python
# tools.py
if __name__ == "__main__":
    import sys, json

    json_path = sys.argv[1]

    with open(json_path, 'r') as f:
        payload = json.load(f)

    function_name = payload["function_name"]
    args = payload.get("args", {})

    if function_name not in globals():
        print(f"Error: Function '{function_name}' not found.")
        exit(1)

    with open("data/sample.html", 'r') as f:
        html_content = f.read()

    func = globals()[function_name]
    result = func(html_content, **args)

    with open("output.html", 'w') as f:
        f.write(result)

    print(f"✅ Executed {function_name}. Result saved to output.html")
```

```bash
python tools.py test_payload.json
```

**When to use:** Any tools.py file during development. Test all tool functions before wiring up the agent. Much faster iteration than running the full Streamlit + ADK loop.

---

## Pattern 10: GOOGLE_GENAI_USE_VERTEXAI Module-Level Env Var

Force Vertex AI mode in the same file where the agent is defined. No need to set it externally.

```python
# agent.py — set BEFORE any google.adk or google.genai imports take effect
import os
os.environ["GOOGLE_GENAI_USE_VERTEXAI"] = "1"

from google.adk.agents import Agent
# ... rest of imports
```

**Note:** This must be set before the SDK makes any authentication decisions. Setting it at the top of agent.py is sufficient — the SDK checks the env var when the agent first runs.

---

## Pattern 11: inspect_code/ Convention

Utility scripts, inspection tools, and debugging scripts live in `inspect_code/`, never in the root directory. Enforced via `.cursorrules`.

```
inspect_code/
├── check_imports.py          # Verify package imports work
├── check_adk_imports.py      # ADK-specific import checks
├── inspect_agent_fields.py   # Introspect agent object fields
├── inspect_gemini.py         # Gemini API exploration
├── test_session.py           # ADK session testing
└── ...
```

```
# .cursorrules
UI Logic must be in main.py, Agent Logic in agent.py.
Inspection and utility scripts (e.g. check_*, inspect_*) MUST be created in the inspect_code/ directory.
```

**Why:** Keeps root clean. These scripts are development tools, not production code. Same intent as `_old/` folders in other repos — keep reference code without polluting the main structure.

---

## Pattern 12: Streamlit Agent Session ID Persistence

Generate agent session ID once per browser session. Pass it on every agent call so the agent remembers conversation history.

```python
# main.py
import uuid

if "agent_session_id" not in st.session_state:
    st.session_state.agent_session_id = str(uuid.uuid4())

# Later, in agent call:
response_text, new_html = agent.run_agent(
    user_input=prompt,
    current_html=st.session_state.current_html,
    session_id=st.session_state.agent_session_id,   # ← always same ID within browser session
)
```

**How it works:** `st.session_state` persists for the duration of a browser tab session. The UUID is generated once on first load and reused. Combined with the global `InMemorySessionService` in `agent.py`, the agent accumulates conversation history within the Streamlit session.

---

## Pattern 13: Sidebar Chat + Main Preview Layout

Standard layout for Streamlit chat + document editor apps.

```python
# main.py
with st.sidebar:
    st.header("Agent Chat")

    # Action buttons at top
    if st.button("Undo Last Change", disabled=len(st.session_state.html_history) == 0):
        undo()

    st.divider()

    # Scrollable chat history container
    chat_container = st.container(height=500)
    with chat_container:
        for msg in st.session_state.chat_history:
            with st.chat_message(msg["role"]):
                st.write(msg["content"])

    # Chat input (pinned to bottom of sidebar)
    if prompt := st.chat_input("Instruct agent..."):
        # ... handle prompt ...

# Main area: tabbed viewer
tab1, tab2 = st.tabs(["Preview", "Code"])
with tab1:
    import streamlit.components.v1 as components
    components.html(st.session_state.current_html, height=800, scrolling=True)
with tab2:
    st.code(st.session_state.current_html, language='html')
```

**When to use:** Any document editor (HTML, markdown, JSON, code) where you want chat on the left and live preview on the right.
