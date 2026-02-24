# Architecture: managed-rag-google-file-search-api-v1

## What This App Is

A **Google Managed RAG tutorial project** — structured as 6 progressive lessons that walk through the complete lifecycle of using Google's File Search API for retrieval-augmented generation.

**Key distinction from crawl4ai-exp:**
- crawl4ai = DIY RAG (you manage chunking + embeddings + Chroma)
- This = **Google Managed RAG** (Google handles chunking, embedding, indexing — you just upload)

---

## System Flow

```
LESSON 1: Check API availability
    └── client.file_search_stores exists?

LESSON 2: Create Store
    └── client.file_search_stores.create()
        └── store_name.txt  ← persisted state

LESSON 3: Upload File
    ├── user picks file from /docs
    ├── client.file_search_stores.upload_to_file_search_store()  (async)
    ├── polling loop: operation.done + client.operations.get()
    └── verify: client.file_search_stores.documents.list()

LESSON 5: Pre-process Document (optional enhancement)
    ├── read PDF as bytes
    ├── types.Part.from_bytes(data=pdf_bytes, mime_type="application/pdf")
    ├── gemini-2.5-flash: create structured summary
    └── save to docs/filename_SUMMARY.txt
        └── then upload summary via Lesson 3

LESSON 4 / query_interactive.py: Query
    └── client.models.generate_content(
            tools=[types.Tool(file_search=types.FileSearch(
                file_search_store_names=[store_name]
            ))]
        )
        └── response.text  (answer)
        └── response.candidates[0].grounding_metadata  (citations)
```

---

## File Structure

```
managed-rag-google-file-search-api-v1/
├── src/
│   ├── lesson_1_check_api.py        ← API availability check
│   ├── lesson_2_create_store.py     ← Create file search store
│   ├── lesson_3_upload_file.py      ← Upload file (async with polling)
│   ├── lesson_4_query.py            ← Query with test questions
│   ├── lesson_5_create_summary.py   ← PDF → structured summary
│   ├── lesson_6_universal_summary.py← Universal summary prompt
│   ├── query_interactive.py         ← Interactive chat loop
│   ├── check_store.py               ← Inspect store contents
│   ├── verify_store.py              ← List all stores in account
│   ├── cleanup_store.py             ← Delete docs (selective or all)
│   ├── test_gemini.py               ← Test model access (Vertex AI)
│   ├── test_auth.py                 ← Test Vertex AI auth
│   ├── utils/
│   │   └── cost_calculator.py       ← Embedding cost estimator
│   └── _old/                        ← Deprecated API surface experiments
│       ├── ingest.py                ← Old: client.files.* API
│       ├── query.py                 ← Old query attempt
│       ├── check_files_api.py       ← Old API exploration
│       └── ...
├── docs/                            ← Upload files go here (user documents)
├── store_name.txt                   ← Persisted store ID (file-based state)
├── requirements.txt                 ← Flat requirements file
└── .gitignore
```

---

## State Management

**Exactly one state file:** `store_name.txt` at project root.

Contains the full resource path: `fileSearchStores/tonyteststore-wsb9ns3bu5m0`

Every script after lesson 2 reads it with:
```python
with open("store_name.txt", "r") as f:
    store_name = f.read().strip()
```

This is the same file-based state pattern seen in VidGen and crawl4ai — confirmed cross-repo pattern.

---

## Two Auth Modes (Same SDK)

The repo shows both approaches:

**Mode 1: API Key (main lessons)**
```python
client = genai.Client(api_key=os.getenv("GEMINI_API_KEY"))
```
- Simpler, no GCP project required
- Used for File Search API
- Works for personal/development use

**Mode 2: Vertex AI (test scripts)**
```python
client = genai.Client(
    vertexai=True,
    project="ninth-potion-455712-g9",
    location="us-central1"
)
```
- Requires GCP project + ADC
- Enterprise/production auth
- Same `client.*` interface as API key mode

---

## API Evolution (Visible in `_old/`)

The old `ingest.py` used a different API surface that no longer works:
```python
# OLD (deprecated):
client.files.list_stores()
client.files.create_store(display_name=..., description=...)
client.files.upload(file_path=..., file_store=...)
```

Current correct API:
```python
# CURRENT:
client.file_search_stores.create(config={'display_name': ...})
client.file_search_stores.upload_to_file_search_store(file=..., file_search_store_name=...)
```

**Lesson:** The `google-genai` SDK API surface changed significantly. Always check SDK version (`google-genai==1.55.0` in this repo).

---

## Lesson Sequence

| Lesson | File | What It Does | Output |
|--------|------|--------------|--------|
| 1 | `lesson_1_check_api.py` | Verify SDK has `file_search_stores` | Console only |
| 2 | `lesson_2_create_store.py` | Create store, save ID | `store_name.txt` |
| 3 | `lesson_3_upload_file.py` | Upload + poll for completion | Document in store |
| 5 | `lesson_5_create_summary.py` | PDF → structured summary | `docs/*_SUMMARY.txt` |
| 6 | `lesson_6_universal_summary.py` | PDF → universal summary | `docs/*_UNIVERSAL_SUMMARY.txt` |
| 4 | `lesson_4_query.py` | Test queries (batch) | Console only |
| - | `query_interactive.py` | Live chat loop | Console only |

---

## Query Flow (What Happens Under the Hood)

```
User question
    ↓
client.models.generate_content(model, contents, config with file_search tool)
    ↓
Gemini receives question
    ↓
File Search tool retrieves relevant chunks from store
    ↓
Gemini synthesizes answer from retrieved chunks
    ↓
response.text  (synthesized answer)
response.candidates[0].grounding_metadata  (which chunks were used)
```

Unlike crawl4ai RAG, there is no explicit `retriever.invoke(query)` step. The retrieval is **implicit** — Gemini decides when to use the tool and what to search for.
