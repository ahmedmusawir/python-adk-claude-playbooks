# GHL API RAG Chatbot — Patterns

## Pattern 1: Dual-Representation Upload (md + SUMMARY)

Each API page gets two files uploaded to the store:

```
outputs/pages/
├── marketplace-...-create-contact.md        ← full raw docs
└── marketplace-...-create-contact_SUMMARY.txt  ← structured metadata
```

The summary is a structured text file with fixed fields:
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
Creates a new contact in GHL...
```

**Why both?** The `.md` has detail (parameters, examples). The `_SUMMARY.txt` has machine-parseable structure (what the master index generator reads). Gemini retrieves from both during queries — gets breadth from summaries, depth from full docs.

---

## Pattern 2: Synthetic Master Index for "List All" Queries

The core RAG weakness: "list all categories" — the answer is spread across 707 chunks. Single retrieval can't aggregate it.

**The fix:** Generate a single synthetic document that aggregates everything:

```python
# master_index_generator_v2.py
master_index = []
master_index.append("📊 STATISTICS:")
master_index.append(f"   • Total Categories: {len(endpoints_by_category)}")
master_index.append(f"   • Total Endpoints: {len(all_endpoints)}")
master_index.append("")
master_index.append("📚 ALL API CATEGORIES (Alphabetical):")
for i, category in enumerate(sorted_categories, 1):
    master_index.append(f"{i:2d}. {category} ({len(endpoints_by_category[category])} endpoints)")

master_index.append("📋 ALL ENDPOINTS BY CATEGORY:")
for category in sorted_categories:
    master_index.append(f"## {category.upper()}")
    for endpoint in sorted(endpoints_by_category[category], key=lambda x: x['name']):
        master_index.append(f"  [{endpoint['method']:6s}] {endpoint['name']}")
        master_index.append(f"   URL: {endpoint['url']}")

# Upload this single file to the same store
OUTPUT_FILE.write_text("\n".join(master_index))
```

Then upload this to the same store. When Gemini searches for "MASTER_INDEX", it retrieves this one file and gets everything.

**Critical:** The system prompt must instruct the LLM to search "MASTER_INDEX" for listing queries. The store alone can't trigger this — you have to tell the LLM explicitly.

---

## Pattern 3: System Prompt as Retrieval Strategy

Not just personality/tone — the system prompt encodes the **search strategy**:

```python
SYSTEM_PROMPT = """You are an expert GHL API documentation assistant.

🔍 CRITICAL SEARCH STRATEGIES:

1. For "list all" or "show me the list" queries:
   - ALWAYS search for "MASTER_INDEX" or "ALL CATEGORIES" FIRST
   - Extract the COMPLETE list from the master index
   - Format as numbered list, alphabetically
   - Include endpoint counts: "Contacts (45 endpoints)"

2. For "how many" queries:
   - Search for "STATISTICS" or "MASTER_INDEX"
   - State the EXACT number clearly

3. For specific endpoint queries (e.g., "create contact"):
   - Search for the endpoint name
   - If not found, try variations: create/add, get/fetch, update/modify
   - Return ONLY the relevant endpoint details

4. If initial search fails, try synonyms automatically
"""
```

**Pattern:** The LLM is using File Search as a tool — it decides what to search. Tell it explicitly what search terms to use for different query patterns. This dramatically improves retrieval quality.

---

## Pattern 4: Filename-Based Category Extraction (v1 vs v2 fix)

**V1 (broken):** LLM parsed category from summary content → generated 290 garbage categories because LLMs hallucinate and over-split.

**V2 (correct):** Parse the filename itself and map to ground truth:

```python
# Filename pattern: marketplace-gohighlevel-com-docs-ghl-{CATEGORY}-{endpoint}_SUMMARY.txt

TRUE_CATEGORIES = {
    'contacts': 'Contacts',
    'contact': 'Contacts',
    'calendars': 'Calendars',
    'calendar': 'Calendars',
    'conversations': 'Conversations',
    # ... 50+ mappings
}

def extract_category_from_filename(filename):
    name = filename.replace('_SUMMARY.txt', '')
    parts = name.split('-')

    try:
        ghl_index = parts.index('ghl')
        category_part = parts[ghl_index + 1]  # segment right after 'ghl'
        return TRUE_CATEGORIES.get(category_part.lower(), "Other")
    except ValueError:
        return "Other"
```

**Rule:** Never trust LLM-extracted metadata for structured classification. If the structure is in the filename, parse the filename.

---

## Pattern 5: Batch Upload with Per-File Error Handling

```python
successful = 0
failed = 0
start_time = time.time()

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

        # Poll for completion
        while not operation.done:
            time.sleep(0.5)
            operation = client.operations.get(operation)

        successful += 1

        # Progress every 50 files
        if i % 50 == 0:
            elapsed = time.time() - start_time
            rate = i / elapsed
            remaining = (total_count - i) / rate
            print(f"Progress: {i}/{total_count} | ~{remaining / 60:.1f} min remaining")

    except Exception as e:
        print(f"❌ Error: {e}")
        failed += 1
        continue  # Never stop the batch for one failure
```

Key points:
- `continue` on failure — one bad file doesn't kill the batch
- Progress every 50 files with rate + estimated remaining time
- Dual mime type: `.md` → `text/markdown`, `.txt` → `text/plain`
- Always poll (`while not operation.done`) — upload is async

---

## Pattern 6: Targeted Document Deletion with force=True

```python
# Find documents to delete
master_index_docs = [
    d for d in docs
    if 'MASTER_INDEX' in d.display_name.upper()
]

# Delete with force=True — required for chunked documents
for doc in master_index_docs:
    client.file_search_stores.documents.delete(
        name=doc.name,
        config={'force': True}  # ← ALWAYS required
    )
```

**Pattern:** Filter by display_name, then delete only what matches. This is the "surgical update" pattern — update the master index without touching the 1414 other documents.

**CRITICAL:** `force=True` is always required when deleting File Search documents. Without it, chunked documents fail silently or raise an error.

---

## Pattern 7: Streamlit Session State Init Block

```python
# Initialize once, use everywhere — idempotent init guards
if 'messages' not in st.session_state:
    st.session_state.messages = []

if 'store_name' not in st.session_state:
    store_file = Path("ghl_store_name.txt")
    if store_file.exists():
        st.session_state.store_name = store_file.read_text().strip()
    else:
        st.error("Store not found!")
        st.stop()  # Halt execution — no store, no app

if 'client' not in st.session_state:
    try:
        st.session_state.client = genai.Client(api_key=os.getenv("GEMINI_API_KEY"))
    except Exception as e:
        st.error(f"Failed to initialize: {e}")
        st.stop()
```

**Pattern:** Three-guard init. Check files first (fail fast if missing), then external services. Use `st.stop()` to halt cleanly rather than letting the app run in a broken state.

---

## Pattern 8: Split-Panel Streamlit Layout

```python
left_col, right_col = st.columns([1, 2])

with left_col:
    st.markdown("### 💬 Chat")
    chat_container = st.container(height=400)  # Fixed height, scrollable
    with chat_container:
        for msg in st.session_state.messages:
            preview = msg['content'][:80] + "..." if len(msg['content']) > 80 else msg['content']
            st.markdown(f"**🤖 Bot:** {preview}")

    prompt = st.chat_input("Ask about GHL API...")

with right_col:
    st.markdown("### 📄 Response")
    response_container = st.container(height=600)
    with response_container:
        st.markdown(st.session_state.current_response)  # Full response, markdown rendered
```

**Key:** `st.container(height=N)` creates a fixed-height scrollable area. Left panel stores compressed history (80-char preview). Right panel displays the full current response. This is the "Lovable/Replit style" split-panel pattern.

---

## Pattern 9: File Search Query — Core Pattern

```python
response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=full_query,  # system prompt + user question concatenated
    config=types.GenerateContentConfig(
        tools=[
            types.Tool(
                file_search=types.FileSearch(
                    file_search_store_names=[store_name]  # full resource path
                )
            )
        ]
    )
)
answer = response.text
```

Simple and clean. The complexity is in `contents` (system prompt prepended) and `store_name` (resource path, not just a name).

---

## Pattern 10: Clickable Example Queries in Streamlit

```python
examples = [
    "List all API categories",
    "Show contact endpoints",
    "How to create contact?",
]

for example in examples:
    if st.button(example, key=f"ex_{example}", use_container_width=True):
        st.session_state.messages.append({"role": "user", "content": example})
        # ... process query
        st.rerun()
```

**Pattern:** Use unique `key` with prefix (avoids Streamlit duplicate key error). `use_container_width=True` makes buttons full-width. `st.rerun()` after state change forces immediate UI refresh.

---

## Summary: The "Doc Site to Chatbot" Blueprint

```
1. Crawl + generate summaries (dual representation)
2. Build master index (aggregated reference for list queries)
3. Upload all to File Search store
4. Persist store ID to text file
5. Build chatbot with system prompt that encodes search strategy
```

Total scripts in the pipeline: **12 utility scripts** + **2 UI files** + **3 discovery files** + **3 crawler files** = very modular, each script does one thing.
