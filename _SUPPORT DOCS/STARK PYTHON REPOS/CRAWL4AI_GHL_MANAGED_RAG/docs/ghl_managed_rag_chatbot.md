# GHL Managed RAG Chatbot — Deep-Dive Reference

## What This Guide Covers

The complete "doc site → chatbot" workflow using Google's File Search API. This is the **fastest path from raw documentation to a working chatbot** — no embedding setup, no vector store, no retrieval tuning.

---

## The Complete Workflow

### Step 1: Create the Store

```python
from google import genai
from pathlib import Path
import os

client = genai.Client(api_key=os.getenv("GEMINI_API_KEY"))

store = client.file_search_stores.create(
    config={'display_name': 'my-docs-store'}
)

# Persist the resource path — all future scripts need this
Path("store_name.txt").write_text(store.name)
print(f"Store created: {store.name}")
# Output: fileSearchStores/my-docs-store-{hash}
```

**Critical:** `store.name` is the full resource path, not just a display name. Always save it.

---

### Step 2: Generate Summaries

For each crawled `.md` page, generate a structured `_SUMMARY.txt`:

```
ENDPOINT NAME:
Create Contact

HTTP METHOD:
POST

ENDPOINT URL:
/contacts/

AUTHENTICATION:
OAuth Access Token or Private Integration Token

DESCRIPTION:
Creates a new contact in the specified location.
```

**Why:** Summaries are machine-parseable. The master index generator reads them without LLM calls. Gemini also retrieves them — they're compact and keyword-rich.

---

### Step 3: Build the Master Index

**The single most important step for list/count queries.**

The problem: "list all API categories" requires aggregating 700+ documents. RAG retrieval alone can't do this.

The solution: generate one document that contains everything:

```python
from pathlib import Path
from collections import defaultdict

# TRUE_CATEGORIES maps filename segments to canonical names
TRUE_CATEGORIES = {
    'contacts': 'Contacts',
    'calendars': 'Calendars',
    'conversations': 'Conversations',
    # ... add all your categories
}

def extract_category_from_filename(filename):
    """Parse category from URL-derived filename."""
    parts = filename.replace('_SUMMARY.txt', '').split('-')
    # Pattern: ...-docs-ghl-{CATEGORY}-{endpoint}
    try:
        ghl_idx = parts.index('ghl')
        category_part = parts[ghl_idx + 1]
        return TRUE_CATEGORIES.get(category_part.lower(), "Other")
    except ValueError:
        return "Other"

# Build index from summaries
endpoints_by_category = defaultdict(list)
summary_files = [f for f in Path("outputs/pages").glob("*_SUMMARY.txt")
                 if 'MASTER_INDEX' not in f.name]

for sf in summary_files:
    category = extract_category_from_filename(sf.name)
    if category == "Other":
        continue
    endpoint_data = parse_summary(sf)  # extract name, method, url
    if endpoint_data:
        endpoints_by_category[category].append(endpoint_data)

# Generate the index document
lines = []
lines.append("📊 STATISTICS:")
lines.append(f"   • Total Categories: {len(endpoints_by_category)}")
lines.append(f"   • Total Endpoints: {sum(len(v) for v in endpoints_by_category.values())}")
lines.append("")
lines.append("📚 ALL API CATEGORIES (Alphabetical):")
for i, cat in enumerate(sorted(endpoints_by_category.keys()), 1):
    lines.append(f"{i:2d}. {cat} ({len(endpoints_by_category[cat])} endpoints)")

Path("outputs/GHL_API_MASTER_INDEX.txt").write_text("\n".join(lines))
```

**Rule:** Build the index from filenames, not from LLM-extracted categories. Filenames are deterministic; LLM extraction is not.

---

### Step 4: Batch Upload

```python
import time
from pathlib import Path

store_name = Path("store_name.txt").read_text().strip()
pages_dir = Path("outputs/pages")

# Pair each .md with its _SUMMARY.txt
all_files = []
for md in sorted(pages_dir.glob("*.md")):
    all_files.append(md)
    summary = pages_dir / f"{md.stem}_SUMMARY.txt"
    if summary.exists():
        all_files.append(summary)

# Upload master index too
master_index = Path("outputs/GHL_API_MASTER_INDEX.txt")
if master_index.exists():
    all_files.append(master_index)

# Batch upload with per-file error handling
successful = 0
failed = 0

for i, file in enumerate(all_files, 1):
    try:
        mime_type = 'text/plain' if file.suffix == '.txt' else 'text/markdown'

        operation = client.file_search_stores.upload_to_file_search_store(
            file=str(file),
            file_search_store_name=store_name,
            config={
                'display_name': file.stem,
                'mime_type': mime_type
            }
        )

        # Upload is async — must poll
        while not operation.done:
            time.sleep(0.5)
            operation = client.operations.get(operation)

        successful += 1

    except Exception as e:
        print(f"Failed: {file.name} — {e}")
        failed += 1
        continue  # never stop the batch

print(f"Done: {successful} uploaded, {failed} failed")
```

---

### Step 5: Build the Chatbot Query Function

```python
from google import genai
from google.genai import types
import os

client = genai.Client(api_key=os.getenv("GEMINI_API_KEY"))
store_name = open("store_name.txt").read().strip()

SYSTEM_PROMPT = """You are a documentation assistant.

SEARCH STRATEGIES:
- For "list all" queries → search for "MASTER_INDEX" first
- For "how many" queries → search for "STATISTICS"
- For specific items → search by name, try variations if not found
- If search fails → try synonyms before giving up
"""

def query_docs(question: str) -> str:
    full_query = f"{SYSTEM_PROMPT}\n\nUser Question: {question}"

    response = client.models.generate_content(
        model="gemini-2.5-flash",
        contents=full_query,
        config=types.GenerateContentConfig(
            tools=[
                types.Tool(
                    file_search=types.FileSearch(
                        file_search_store_names=[store_name]
                    )
                )
            ]
        )
    )
    return response.text
```

---

## Streamlit Chatbot Template

### Simple Version (sidebar + chat)

```python
import streamlit as st
from pathlib import Path
import os

# Initialize once
if 'messages' not in st.session_state:
    st.session_state.messages = []

if 'store_name' not in st.session_state:
    store_file = Path("store_name.txt")
    if not store_file.exists():
        st.error("Store not found!")
        st.stop()
    st.session_state.store_name = store_file.read_text().strip()

if 'client' not in st.session_state:
    st.session_state.client = genai.Client(api_key=os.getenv("GEMINI_API_KEY"))

# Chat UI
for msg in st.session_state.messages:
    with st.chat_message(msg["role"]):
        st.markdown(msg["content"])

if prompt := st.chat_input("Ask a question..."):
    st.session_state.messages.append({"role": "user", "content": prompt})
    with st.chat_message("user"):
        st.markdown(prompt)

    with st.chat_message("assistant"):
        with st.spinner("Thinking..."):
            answer = query_docs(prompt)
            st.markdown(answer)
            st.session_state.messages.append({"role": "assistant", "content": answer})
```

### Split-Panel Version (for long responses)

```python
left_col, right_col = st.columns([1, 2])

if 'current_response' not in st.session_state:
    st.session_state.current_response = "Ask me anything!"

with left_col:
    st.markdown("### 💬 Chat")

    # Scrollable history with previews
    chat_container = st.container(height=400)
    with chat_container:
        for msg in st.session_state.messages:
            if msg["role"] == "user":
                st.markdown(f"**You:** {msg['content']}")
            else:
                preview = msg['content'][:80] + "..." if len(msg['content']) > 80 else msg['content']
                st.markdown(f"**Bot:** {preview}")

    prompt = st.chat_input("Ask...")
    if prompt:
        st.session_state.messages.append({"role": "user", "content": prompt})
        with st.spinner("Thinking..."):
            answer = query_docs(prompt)
            st.session_state.messages.append({"role": "assistant", "content": answer})
            st.session_state.current_response = answer
        st.rerun()

with right_col:
    st.markdown("### 📄 Response")
    with st.container(height=600):
        st.markdown(st.session_state.current_response)
```

**When to use split-panel:** When responses are long (tables, code blocks). When users need a reading area, not just chat bubbles.

---

## Document Lifecycle Management

### Check what's in the store

```python
docs = list(client.file_search_stores.documents.list(parent=store_name))
print(f"Total documents: {len(docs)}")

md_docs = [d for d in docs if not d.display_name.endswith('_SUMMARY')]
summary_docs = [d for d in docs if d.display_name.endswith('_SUMMARY')]
print(f"Markdown: {len(md_docs)}, Summaries: {len(summary_docs)}")
```

### Delete specific documents (surgical update)

```python
# Find by display_name pattern
target_docs = [d for d in docs if 'MASTER_INDEX' in d.display_name.upper()]

for doc in target_docs:
    client.file_search_stores.documents.delete(
        name=doc.name,
        config={'force': True}  # ← ALWAYS required for chunked docs
    )
    print(f"Deleted: {doc.display_name}")
```

### The update workflow

```
1. Delete old master index (delete_old_master_index.py)
2. Regenerate master index (master_index_generator_v2.py)
3. Upload new master index (upload_master_index_v2.py)
4. Individual docs: same delete → re-upload pattern
```

---

## Gotchas

### 1. force=True is mandatory for deletion
```python
# ❌ Will fail for chunked documents
client.file_search_stores.documents.delete(name=doc.name)

# ✅ Always use force=True
client.file_search_stores.documents.delete(name=doc.name, config={'force': True})
```

### 2. store.name is a full resource path
```
# It looks like this:
fileSearchStores/ghlapiv2docs-snonl43ig3vp

# NOT just:
ghlapiv2docs-snonl43ig3vp
```
Every API call needs the full path.

### 3. Upload is async — must poll
```python
operation = client.file_search_stores.upload_to_file_search_store(...)
while not operation.done:          # ← MUST poll
    time.sleep(0.5)
    operation = client.operations.get(operation)
```
Checking `operation.done` once and moving on = silent failure.

### 4. Bulk delete is not available
There's no "delete all documents" API. To reset a store: delete the store, create a new one. Or iterate over all documents and delete individually. This takes time for large stores.

### 5. LLM category extraction = don't do it
Using an LLM to extract structured metadata from your own filenames produces unreliable results. Parse the filename pattern directly. Ground truth mapping > LLM parsing for structured data.

### 6. "Session memory" in Streamlit ≠ conversational memory
`st.session_state.messages` persists chat history in the UI. But if you're not passing the history in each Gemini request, the LLM doesn't remember prior turns. For pure lookup bots, this is fine. For conversational assistants, you must include history in `contents`.

---

## Cost Model

| Operation | Cost |
|-----------|------|
| Store creation | Free |
| Document storage | Free |
| Upload (embedding) | One-time cost per document |
| Query | Per token (generation) |

For ~1414 files totaling ~10-50MB: upload cost is very low (cents). The main ongoing cost is query tokens.

---

## The "Doc Site to Chatbot" Checklist

```
□ Crawl site → .md files
□ Generate structured _SUMMARY.txt for each .md
□ Build master index (filename-based categorization)
□ Create File Search store
□ Save store resource path to store_name.txt
□ Batch upload: all .md + all _SUMMARY.txt + master index
□ Write system prompt with explicit search strategies
□ Build chatbot (CLI or Streamlit)
□ Test: "list all", "how many", specific endpoint
```

---

## When to Use Managed RAG vs DIY

| Factor | Managed (File Search) | DIY (LangChain + Chroma) |
|--------|----------------------|--------------------------|
| Setup time | Minutes | Hours |
| Retrieval control | None | Full (chunk_size, MMR, etc.) |
| Vendor lock-in | Google only | Provider-agnostic |
| Embedding cost | One-time at upload | Every reindex |
| Local storage | None | Disk (vector store) |
| Debugging retrieval | Black box | Inspectable |
| Production suitability | Yes (simple) | Yes (tunable) |

**Use managed when:** Speed matters, vendor lock-in is OK, retrieval quality is good enough.
**Use DIY when:** You need to tune retrieval, stay vendor-agnostic, or inspect what's being retrieved.
