# BeautifulSoup HTML Editing Integration

Deep dive on using BeautifulSoup as the execution layer for AI-driven HTML manipulation.

---

## Why BeautifulSoup as an AI Tool Layer

The core insight in this repo: **let the AI interpret intent, let BeautifulSoup execute changes**.

| Approach | Problem |
|----------|---------|
| LLM outputs full HTML | Hallucination, truncation, structural corruption on large templates |
| LLM outputs regex patterns | Brittle, breaks on nested HTML, can't handle attribute complexity |
| JavaScript DOM manipulation | Requires a browser context or headless browser |
| **BeautifulSoup tools** | Deterministic, CSS-selector based, works on any HTML string |

BeautifulSoup operates on HTML strings in Python — no browser required. It parses the document, lets you find elements with CSS selectors, and mutates them precisely.

---

## The Tool Contract

Every tool in this pattern follows the same contract:

```python
def tool_name(selector: str, ..., html_content: str = None) -> str:
    global _active_html
    if html_content is None:
        html_content = _active_html

    soup = BeautifulSoup(html_content, 'html.parser')
    elements = soup.select(selector)

    # ... mutation logic ...

    result = str(soup)
    _active_html = result  # write back to global state
    return result
```

Key points:
- `html_content=None` default enables both modes: ADK agent mode (uses global state) and CLI/test mode (pass HTML directly)
- `soup.select(selector)` handles all CSS selector types
- `str(soup)` serializes the modified DOM back to an HTML string
- Write back to `_active_html` so chained tool calls see cumulative changes

---

## Tool Reference

### `apply_style_edit(selector, css_property, css_value)`

**Purpose:** Change one CSS property in an inline style attribute.

**Key technique:** Parse inline style string → dict → update → reconstruct. Preserves all other properties.

```python
def apply_style_edit(selector: str, css_property: str, css_value: str, html_content: str = None) -> str:
    soup = BeautifulSoup(html_content, 'html.parser')
    elements = soup.select(selector)

    for element in elements:
        current_style = element.get('style', '')
        style_dict = {}

        # Parse: "color: red; font-size: 16px" → {"color": "red", "font-size": "16px"}
        if current_style:
            for style_pair in [s.strip() for s in current_style.split(';') if s.strip()]:
                if ':' in style_pair:
                    key, val = style_pair.split(':', 1)
                    style_dict[key.strip().lower()] = val.strip()

        # Update or add property
        style_dict[css_property.strip().lower()] = css_value.strip()

        # Reconstruct: {"color": "blue", "font-size": "16px"} → "color: blue; font-size: 16px;"
        new_style = '; '.join([f"{k}: {v}" for k, v in style_dict.items()]) + ';'
        element['style'] = new_style

    return str(soup)
```

**Example agent call:**
```
apply_style_edit(selector="a[style*='background']", css_property="background-color", css_value="#0066FF")
```

---

### `update_text_content(selector, new_text)`

**Purpose:** Replace all text inside an element.

```python
def update_text_content(selector: str, new_text: str, html_content: str = None) -> str:
    soup = BeautifulSoup(html_content, 'html.parser')
    elements = soup.select(selector)

    for element in elements:
        element.string = new_text   # replaces all content with plain text

    return str(soup)
```

**Limitation:** `element.string = ...` only sets plain text. If the element has child tags, this can break them. Use `update_inner_html` for elements with nested markup.

---

### `update_inner_html(selector, new_html)`

**Purpose:** Replace the inner content of an element with arbitrary HTML (including nested tags).

```python
def update_inner_html(selector: str, new_html: str, html_content: str = None) -> str:
    soup = BeautifulSoup(html_content, 'html.parser')
    elements = soup.select(selector)

    for element in elements:
        element.clear()  # remove existing children

        # Parse the new HTML fragment
        new_soup = BeautifulSoup(new_html, 'html.parser')

        # Handle body wrapping (BeautifulSoup wraps fragments in <body>)
        if new_soup.body:
            tags_to_insert = list(new_soup.body.contents)
        else:
            tags_to_insert = list(new_soup.contents)

        for tag in tags_to_insert:
            element.append(tag)

    return str(soup)
```

**Example:** Turn "Click here" into "Click <span style='color:red'>here</span>"

---

### `insert_element_relative(target_selector, new_html, position)`

**Purpose:** Insert new HTML elements relative to an existing element.

**Positions:** `before`, `after`, `inside_start`, `inside_end`

```python
def insert_element_relative(target_selector: str, new_html: str, position: str = 'after', html_content: str = None) -> str:
    soup = BeautifulSoup(html_content, 'html.parser')
    target_elements = soup.select(target_selector)

    if not target_elements:
        return f"ERROR: No element found matching selector '{target_selector}'"

    for target in target_elements:
        temp_soup = BeautifulSoup(new_html, 'html.parser')
        tags = list(temp_soup.body.contents) if temp_soup.body else list(temp_soup.contents)

        if position == 'before':
            for tag in tags:
                target.insert_before(tag)
        elif position == 'after':
            for tag in reversed(tags):   # ← reversed to maintain order
                target.insert_after(tag)
        elif position == 'inside_start':
            for tag in reversed(tags):
                target.insert(0, tag)
        elif position == 'inside_end':
            for tag in tags:
                target.append(tag)

    return str(soup)
```

**Note:** `insert_after` inserts in reverse order when inserting multiple tags — `reversed()` corrects the order.

**Error handling:** Returns `"ERROR: ..."` string (not raises) when selector matches nothing. Agent instruction says: "If a tool fails (returns 'ERROR'), try a different selector."

---

### `remove_element(selector)`

**Purpose:** Delete all elements matching a selector.

```python
def remove_element(selector: str, html_content: str = None) -> str:
    soup = BeautifulSoup(html_content, 'html.parser')
    elements = soup.select(selector)

    if not elements:
        return "ERROR: No elements found to remove."

    for element in elements:
        element.decompose()   # ← destroys the element and its children

    return str(soup)
```

**`decompose()` vs `extract()`:**
- `decompose()` — destroys the element permanently (can't reuse it)
- `extract()` — removes and returns the element (can reinsert elsewhere)
- For a delete-only tool, `decompose()` is correct

---

## CSS Selector Patterns for HTML Emails

HTML email templates use inline styles heavily. These selectors are what the agent learns to use:

| Target | Selector Example |
|--------|-----------------|
| Elements by tag | `h1`, `p`, `a`, `table`, `td` |
| Elements with specific style | `a[style*='background']` |
| Elements by class | `.button`, `.header-text` |
| First/only element | `h1:first-child` |
| Specific attribute value | `td[align='center']` |
| Nested elements | `table td a` |

**Why attribute style selectors work for email HTML:**
GHL email templates don't use CSS classes — they use inline styles. `a[style*='red']` finds anchors whose style attribute contains "red". This is the primary selector strategy for email HTML manipulation.

---

## Error Handling Philosophy

Tools return error strings, not exceptions:

```python
# Good: return error string
if not elements:
    return f"ERROR: No element found matching selector '{target_selector}'"

# Never: raise exception from a tool
# raise ValueError(f"No element found")  ← would break agent session
```

Agent instruction includes: `"If a tool fails (returns 'ERROR'), try a different selector or strategy."` — the agent can adapt without a session crash.

This matches the ADK MCP pattern (from ghl-adk-mcp-server-v1): never throw from a tool handler; always return error as a value.

---

## Global State Flow (Chain of Tool Calls)

When the agent needs to make multiple changes (e.g., "change button color AND update button text"), it chains tool calls. Each call sees the state left by the previous:

```
User: "Change button to blue and update text to 'Buy Now'"
    ↓
Agent calls: apply_style_edit(selector="a.button", css_property="background-color", css_value="blue")
    _active_html = [HTML with blue button]

Agent calls: update_text_content(selector="a.button", new_text="Buy Now")
    _active_html = [HTML with blue button AND "Buy Now" text]

agent.py reads: tools.get_active_html()
    → returns [HTML with both changes applied]
```

No coordination required from the agent — the global state automatically chains.

---

## Testing Approach

### JSON Test Harness (No Agent Required)

```bash
# Create payload
cat > test_style.json << 'EOF'
{
  "function_name": "apply_style_edit",
  "args": {
    "selector": "h1",
    "css_property": "color",
    "css_value": "#FF0000"
  }
}
EOF

# Run
python tools.py test_style.json

# Check result
open output.html  # or view in browser
```

Test all tools before wiring into the agent. Verify selectors work against your actual HTML template.

### Print Logging

Every tool call logs to stdout:
```python
print(f"🛠️ [TOOL EXEC] apply_style_edit called with args: {locals()}")
```

Visible in terminal when running `streamlit run main.py`. Confirms what the agent is actually calling.

---

## Parser Choice: `html.parser` vs `lxml`

```python
soup = BeautifulSoup(html_content, 'html.parser')   # built-in, no extra dep
# vs
soup = BeautifulSoup(html_content, 'lxml')           # faster, needs lxml installed
```

`html.parser` is used here (in requirements.txt: only `beautifulsoup4`, no `lxml`). Sufficient for typical email templates. If parsing performance becomes an issue with large templates, add `lxml` as a dependency.

---

## BeautifulSoup Gotchas for HTML Email

1. **`str(soup)` adds DOCTYPE** — For email HTML fragments that don't have `<!DOCTYPE>`, BeautifulSoup may add one. Not a problem for GHL templates which are full HTML documents.

2. **Tag objects are shared** — When you extract tags from one soup object and append to another, the object is moved (not copied). Always create a new soup for each fragment parse.

3. **`element.string` fails on elements with child tags** — `element.string` returns `None` if the element has children. Use `update_inner_html` instead of `update_text_content` for elements with nested markup.

4. **CSS `*=` attribute selector** — `[style*='value']` is a substring match. Reliable for finding elements by partial style content (e.g., `[style*='background']`).

5. **`decompose()` is irreversible on the soup object** — The element is destroyed. If you need undo, always operate on a fresh BeautifulSoup parse of the pre-mutation HTML (the undo stack in main.py handles this at the application level).
