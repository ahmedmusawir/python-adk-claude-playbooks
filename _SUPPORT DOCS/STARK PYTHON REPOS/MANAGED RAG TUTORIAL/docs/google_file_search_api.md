# Google File Search API — Integration Deep-Dive

## What Is It

Google's **File Search API** is a managed RAG service inside the `google-genai` SDK. You upload documents, Google handles all of the RAG infrastructure (chunking, embedding, indexing), and you query using a `file_search` tool passed to `generate_content`.

**SDK package:** `google-genai` (not `google-cloud-aiplatform`)
**Minimum version:** `google-genai>=1.0.0` (this repo uses 1.55.0)
**Access point:** `client.file_search_stores`

---

## Setup

### Install
```bash
pip install google-genai python-dotenv
```

### Auth (two options)
```python
from google import genai

# Option A: API Key (dev/personal)
client = genai.Client(api_key="YOUR_GEMINI_API_KEY")

# Option B: Vertex AI (production/enterprise)
client = genai.Client(
    vertexai=True,
    project="your-gcp-project-id",
    location="us-central1"
)
```

### Verify API is available
```python
if hasattr(client, 'file_search_stores'):
    methods = [m for m in dir(client.file_search_stores) if not m.startswith('_')]
    # Should show: create, delete, documents, get, list
```

---

## Core Concepts

### Store
A persistent container for documents. Like a "knowledge base" or "namespace".
- Has a `display_name` (human-readable)
- Has a `name` (resource path used in all API calls): `fileSearchStores/storename-randomhash`
- Can hold many documents
- Can query across multiple stores simultaneously
- Storage is **free** — only embedding creation costs money

### Document
A file uploaded to a store.
- Same `name`/`display_name` pattern as stores
- After upload, Google automatically: chunks → embeds → indexes
- Supports: `.txt`, `.pdf` confirmed (other formats may work)
- Cannot be "updated" — delete + re-upload to refresh

### Operation
Async result returned by `upload_to_file_search_store`. Must be polled.

---

## Full Lifecycle Code Reference

### 1. Create Store
```python
store = client.file_search_stores.create(
    config={'display_name': 'My-Knowledge-Base'}
)
# Save for later
store_resource_path = store.name  # "fileSearchStores/my-knowledge-base-abc123"
```

### 2. Upload Document
```python
import time

operation = client.file_search_stores.upload_to_file_search_store(
    file="path/to/document.pdf",            # string or Path
    file_search_store_name=store_resource_path,
    config={'display_name': 'document'}     # optional human name
)

# Poll until complete
while not operation.done:
    time.sleep(1)
    operation = client.operations.get(operation)
# Document is now chunked, embedded, and indexed
```

### 3. Verify Upload
```python
docs = list(client.file_search_stores.documents.list(
    parent=store_resource_path
))
for doc in docs:
    print(f"{doc.display_name}  ({doc.name})")
```

### 4. Query
```python
from google.genai import types

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Your question here",
    config=types.GenerateContentConfig(
        tools=[
            types.Tool(
                file_search=types.FileSearch(
                    file_search_store_names=[store_resource_path]
                    # Can pass multiple: [store1, store2, store3]
                )
            )
        ]
    )
)

print(response.text)  # Synthesized answer

# Get citation metadata
metadata = response.candidates[0].grounding_metadata
```

### 5. Delete Document
```python
# ALWAYS use force=True — without it, deletion fails for processed docs
client.file_search_stores.documents.delete(
    name=doc.name,
    config={'force': True}
)
```

### 6. List All Stores
```python
all_stores = list(client.file_search_stores.list())
```

---

## Common Gotchas

### Gotcha 1: `force=True` Required for Delete
```python
# FAILS for any document that has been processed:
client.file_search_stores.documents.delete(name=doc.name)

# WORKS:
client.file_search_stores.documents.delete(name=doc.name, config={'force': True})
```

### Gotcha 2: SDK API Surface Changed
Old API (no longer works):
```python
client.files.list_stores()
client.files.create_store(display_name=...)
client.files.upload(file_path=..., file_store=...)
```

Current API:
```python
client.file_search_stores.list()
client.file_search_stores.create(config={'display_name': ...})
client.file_search_stores.upload_to_file_search_store(file=..., file_search_store_name=...)
```

### Gotcha 3: Store Name Is a Resource Path
```python
store.name == "fileSearchStores/mystore-randomhash"  # NOT just "mystore"
```
Use `store.name` (full path) in all API calls. Use `store.display_name` only for display.

### Gotcha 4: Upload Is Async
The upload call returns immediately with an operation object. The document is NOT immediately queryable. Must poll `operation.done`.

### Gotcha 5: Retrieval Is Implicit
Unlike LangChain where you call `retriever.invoke(query)` explicitly, File Search retrieval is handled by Gemini internally. Gemini decides when to use the tool and what to search for. You cannot inspect what was retrieved unless you parse `grounding_metadata`.

---

## RAG Quality Enhancement: Summary + Original Strategy

**Problem:** Chunking breaks counting/enumeration questions across multiple chunks.

**Solution:** Pre-generate a structured summary that lists ALL entities, upload alongside original.

```python
# Step 1: Read full PDF
with open("docs/document.pdf", "rb") as f:
    pdf_bytes = f.read()

# Step 2: Generate comprehensive summary
response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=[
        types.Part.from_bytes(data=pdf_bytes, mime_type="application/pdf"),
        """Create a structured summary listing ALL:
        - People mentioned (full names)
        - Companies/Organizations (ALL of them, including brief mentions)
        - Dates and numbers
        - Key topics and keywords
        Be EXHAUSTIVE — this will be used for search."""
    ]
)

# Step 3: Save summary
summary_path = Path("docs/document_SUMMARY.txt")
summary_path.write_text(response.text)

# Step 4: Upload BOTH to the store
# upload original... (lesson 3 pattern)
# upload summary...  (lesson 3 pattern)
```

**Why it works:**
- Original doc: great for specific detail questions
- Summary doc: great for "list all" and "how many" questions
- Gemini retrieves from both, synthesizes best answer

---

## Cost Model

| Operation | Cost |
|-----------|------|
| Create store | Free |
| Store storage | Free (indefinite) |
| Embedding (upload) | ~$0.00015 per 1,000 tokens (~4 chars/token) |
| Query (generate_content) | Standard Gemini pricing |
| Delete | Free |

**Rough calculation:**
- 100KB text file ≈ 25,000 tokens ≈ $0.00375 to embed
- PDF embedding may differ (depends on content density)
- Queries billed by input+output tokens of the generate_content call

---

## Multi-Store Queries

```python
# Query across multiple knowledge bases simultaneously
response = client.models.generate_content(
    model="gemini-2.5-pro",
    contents=question,
    config=types.GenerateContentConfig(
        tools=[types.Tool(
            file_search=types.FileSearch(
                file_search_store_names=[
                    store_a_resource_path,
                    store_b_resource_path,
                    store_c_resource_path,
                ]
            )
        )]
    )
)
```

Use case: "client docs" + "internal policies" + "product specs" in one query.

---

## Model Choice for Queries

| Model | Use When |
|-------|----------|
| `gemini-2.5-flash` | Batch queries, cost-sensitive, simple Q&A |
| `gemini-2.5-pro` | Interactive chat, complex reasoning, multi-step questions |

Both models support the `file_search` tool identically.

---

## Comparison: File Search API vs DIY RAG (LangChain + Chroma)

| Aspect | File Search API | LangChain + Chroma |
|--------|----------------|---------------------|
| Setup | 3 API calls | ~50 lines of config |
| Chunking control | None (Google decides) | Full (chunk_size, overlap) |
| Embedding model | Google text-embedding-005 | Any (OpenAI, Vertex, local) |
| Retrieval method | Gemini-decided (implicit) | Explicit (`retriever.invoke()`) |
| Local deployment | No | Yes |
| Vendor lock-in | Google API | Framework-agnostic |
| Debugging retrieval | Limited (grounding_metadata) | Full visibility |
| Cost transparency | Opaque | Transparent |
| Speed to working RAG | ~10 minutes | ~1-2 hours |

**Rule of thumb:**
- Prototype in 10 min → File Search API
- Production with control needs → LangChain + Chroma (or similar)
- Production on Google Cloud, simplicity matters → File Search API
