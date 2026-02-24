# Patterns: managed-rag-google-file-search-api-v1

## Pattern 1: Google File Search API — Client Setup

```python
from google import genai
from dotenv import load_dotenv
import os

load_dotenv()

# API Key mode (personal/dev)
client = genai.Client(api_key=os.getenv("GEMINI_API_KEY"))

# Vertex AI mode (enterprise/production)
client = genai.Client(
    vertexai=True,
    project=os.getenv("GCP_PROJECT"),
    location="us-central1"
)
```

**Same client interface, two auth modes.** All `client.file_search_stores.*` calls work identically with either mode.

---

## Pattern 2: Create File Search Store

```python
store = client.file_search_stores.create(
    config={'display_name': 'My-Store-Name'}
)

# store.name  → full resource path: "fileSearchStores/my-store-abc123"
# store.display_name  → human-readable name

# Persist for later use
with open("store_name.txt", "w") as f:
    f.write(store.name)
```

**Note:** `store.name` is NOT a plain string — it's a resource path like `fileSearchStores/tonyteststore-wsb9ns3bu5m0`. This full path is what every other API call requires.

---

## Pattern 3: Upload File (Async with Polling)

```python
import time

operation = client.file_search_stores.upload_to_file_search_store(
    file=str(file_path),                        # string path
    file_search_store_name=store_name,          # full resource path
    config={'display_name': file_path.stem}     # name without extension
)

# Poll for completion
while not operation.done:
    print(".", end="", flush=True)
    time.sleep(1)
    operation = client.operations.get(operation)

print(" DONE!")
```

**Upload is async.** Returns an operation, not a result. Must poll `operation.done` + refresh with `client.operations.get(operation)`.

What Google does automatically:
1. Chunks the document (splits into semantic pieces)
2. Creates embeddings (vectors)
3. Indexes for fast retrieval

---

## Pattern 4: Query with File Search Tool

```python
from google.genai import types

response = client.models.generate_content(
    model="gemini-2.5-flash",   # or gemini-2.5-pro for better quality
    contents=question,
    config=types.GenerateContentConfig(
        tools=[
            types.Tool(
                file_search=types.FileSearch(
                    file_search_store_names=[store_name]  # list of store paths
                )
            )
        ]
    )
)

answer = response.text

# Access grounding/citation metadata
if hasattr(response.candidates[0], 'grounding_metadata'):
    metadata = response.candidates[0].grounding_metadata
```

**Retrieval is implicit** — Gemini decides when/what to search. No explicit `.invoke(query)` step like LangChain retriever.

You can pass **multiple stores** in `file_search_store_names` — query spans all of them.

---

## Pattern 5: List Store Contents

```python
# Using 'parent' parameter (primary method in lessons)
docs = list(client.file_search_stores.documents.list(parent=store_name))

# OR using named parameter (used in verify_store.py)
docs = list(client.file_search_stores.documents.list(
    file_search_store_name=store.name
))

for doc in docs:
    print(f"  {doc.display_name}")  # human name
    print(f"  {doc.name}")          # resource path
```

---

## Pattern 6: Delete Document (Force Flag Required)

```python
# WRONG — fails if document has been chunked:
client.file_search_stores.documents.delete(name=doc.name)

# CORRECT — force=True removes chunks too:
client.file_search_stores.documents.delete(
    name=doc.name,
    config={'force': True}
)
```

**Critical:** Always use `config={'force': True}` when deleting. Without it, deletion fails for any document that has been processed (chunked). This was discovered and fixed in `cleanup_store.py`.

---

## Pattern 7: List All Stores in Account

```python
stores = list(client.file_search_stores.list())

for store in stores:
    print(f"{store.display_name}  →  {store.name}")
    docs = list(client.file_search_stores.documents.list(
        file_search_store_name=store.name
    ))
    print(f"  {len(docs)} documents")
```

---

## Pattern 8: PDF Direct Analysis (No Upload Required)

For pre-processing / generating summaries before upload:

```python
from google.genai import types

with open("docs/document.pdf", "rb") as f:
    pdf_bytes = f.read()

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=[
        types.Part.from_bytes(
            data=pdf_bytes,
            mime_type="application/pdf"
        ),
        "Your prompt here as plain string"  # ✅ Just pass string, not Part
    ]
)

summary_text = response.text
```

**Use case:** Generate a structured summary of the full document, then upload the summary text as a supplementary file. The summary helps RAG answer "how many" / "list all" questions accurately (overcomes chunking limitations for counting tasks).

---

## Pattern 9: Interactive Query Loop

```python
while True:
    question = input("❓ Your question: ").strip()

    if question.lower() in ['exit', 'quit', 'q']:
        break

    if not question:
        continue

    response = client.models.generate_content(
        model="gemini-2.5-pro",  # Pro for better reasoning in interactive mode
        contents=question,
        config=types.GenerateContentConfig(
            tools=[types.Tool(
                file_search=types.FileSearch(
                    file_search_store_names=[store_name]
                )
            )]
        )
    )

    print(response.text)
```

---

## Pattern 10: Store Name as Persistent State (store_name.txt)

```python
# Write (create store)
with open("store_name.txt", "w") as f:
    f.write(store.name)  # e.g., "fileSearchStores/mystore-abc123"

# Read (all subsequent scripts)
with open("store_name.txt", "r") as f:
    store_name = f.read().strip()  # .strip() is important
```

Same file-based state pattern as VidGen (`project_state.json`) and crawl4ai (`site_config.json`). Confirmed cross-repo pattern #1.

---

## Pattern 11: Cost Estimation Utility

```python
def calculate_embedding_cost(file_size_bytes):
    chars_per_token = 4                       # English text approximation
    estimated_tokens = file_size_bytes / chars_per_token
    cost_per_1k_tokens = 0.00015              # Google text-embedding-005 pricing
    embedding_cost = (estimated_tokens / 1000) * cost_per_1k_tokens
    return {
        'estimated_tokens': int(estimated_tokens),
        'embedding_cost': round(embedding_cost, 6),
        'storage_cost': 0.0  # Storage is FREE
    }
```

**Key insight:** Storage in File Search stores is free. Only embedding creation costs money (at upload time, not query time).

---

## Pattern 12: Universal Summary Prompt (Document-Agnostic)

```python
universal_prompt = """
Analyze this document and create a STRUCTURED SUMMARY:

1. DOCUMENT TYPE: (What kind of document is this?)

2. MAIN TOPICS: (What are the 3-5 key topics covered?)

3. KEY ENTITIES - LIST ALL:
   - People mentioned (full names)
   - Companies/Organizations mentioned (ALL of them)
   - Important dates (when available)
   - Important numbers/metrics (when available)

4. MAIN POINTS: (Summarize the key takeaways in bullet points)

5. SEARCHABLE KEYWORDS: (List important terms someone might search for)

Be EXHAUSTIVE with entities - list EVERY company, person, date mentioned.
This summary will be used for search and retrieval, so completeness matters.
"""
```

**Why this matters:** Standard RAG chunking misses counting/listing questions ("how many companies?") because the answer is spread across chunks. Pre-generating a structured summary that lists ALL entities solves this — upload both the original + the summary to the store.
