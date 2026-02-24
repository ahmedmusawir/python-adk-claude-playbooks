# GHL API RAG Chatbot — Architecture

## What This Is

A complete **docs-site-to-chatbot pipeline** for the GoHighLevel (GHL) API documentation. Given a documentation website, this repo crawls every page, structures the content, uploads it to Google's managed RAG (File Search API), and provides a production-ready chatbot UI to query it.

This is the **first domain-specific chatbot** in the extraction series.

---

## The Full Pipeline

```
GHL API Docs Site (marketplace.gohighlevel.com/docs)
        │
        ▼
1. DISCOVER  (discover_site/)
   sitemap_utils.py → sitemap XML → discovered_pages.json
   smart_discover.py → JS-rendered page fallback
        │
        ▼
2. CRAWL  (smart_crawler/)
   crawl4ai → raw scraped markdown
   data_processing.py → structured *_SUMMARY.txt
   outputs/pages/
     ├── *.md          (raw API docs — ~707 files)
     └── *_SUMMARY.txt (structured metadata — ~707 files)
        │
        ▼
3. BUILD MASTER INDEX  (utils/)
   master_index_generator_v2.py → GHL_API_MASTER_INDEX_V2.txt
   (aggregates all endpoints by category — the "list all" answer)
        │
        ▼
4. UPLOAD  (utils/)
   upload_all_ghl_docs.py      → ~1414 files → File Search Store
   upload_master_index_v2.py   → master index → same store
   ghl_store_name.txt          → persisted store resource path
        │
        ▼
5. QUERY  (root/)
   ghl_chatbot_streamlit.py    → Split-panel Streamlit UI
   ghl_chatbot_streamlit-1.py  → Standard chat Streamlit UI (v1)
   utils/ghl_chatbot_cli.py    → Interactive CLI mode
```

---

## File Structure

```
crawl4ai-exp-project-v1/
├── ghl_chatbot_streamlit.py        ← v2 UI (split panel — main)
├── ghl_chatbot_streamlit-1.py      ← v1 UI (sidebar + chat)
├── ghl_store_name.txt              ← persisted File Search store ID
├── discover_site/
│   ├── sitemap_utils.py            ← XML sitemap parser
│   ├── discover.py                 ← main discovery runner
│   └── smart_discover.py           ← JS fallback
├── smart_crawler/
│   ├── crawler.py                  ← crawl4ai async crawler
│   ├── data_processing.py          ← LangChain + LLM summarizer
│   └── schema.py                   ← Pydantic output models
├── utils/
│   ├── create_ghl_store.py         ← create File Search store
│   ├── upload_all_ghl_docs.py      ← batch upload 1414 files
│   ├── upload_master_index.py      ← upload master index v1
│   ├── upload_master_index_v2.py   ← upload master index v2
│   ├── master_index_generator.py   ← build master index v1 (LLM-parsed)
│   ├── master_index_generator_v2.py← build master index v2 (filename-based)
│   ├── cleanup_master_index.py     ← fix category explosion from v1
│   ├── delete_old_master_index.py  ← targeted store document deletion
│   ├── check_ghl_store.py          ← inspect store contents
│   ├── ghl_chatbot_cli.py          ← CLI query interface
│   ├── test_api_connection.py      ← API health check
│   └── test_single_upload.py       ← upload smoke test
└── outputs/
    └── pages/
        ├── *.md                    ← raw crawled docs (707 files)
        ├── *_SUMMARY.txt           ← structured summaries (707 files)
        ├── GHL_API_MASTER_INDEX_SUMMARY.txt   ← v1 (raw LLM output)
        ├── GHL_API_MASTER_INDEX_CLEAN.txt     ← v1 cleaned
        └── GHL_API_MASTER_INDEX_V2.txt        ← v2 (filename-based, best)
```

---

## What's in the File Search Store

Total: **~1415 documents** in a single store named `ghl-api-v2-docs`.

| Document Type | Count | Purpose |
|--------------|-------|---------|
| `.md` files | ~707 | Full endpoint documentation (parameters, examples) |
| `_SUMMARY.txt` files | ~707 | Structured metadata (endpoint name, method, URL, auth) |
| Master Index | 1 | Aggregated reference for "list all" queries |

**Store resource path format:** `fileSearchStores/{name}-{hash}`
**Persisted in:** `ghl_store_name.txt`

---

## Scale

- **Website:** marketplace.gohighlevel.com/docs
- **~38 API categories** (Contacts, Calendars, Conversations, Payments, etc.)
- **~700 endpoints** across all categories
- **~1415 files** in the File Search store
- **Upload time:** ~20–30 minutes for initial batch

---

## UI Architecture

### v2 (Split Panel) — `ghl_chatbot_streamlit.py`
```
┌─────────────────────┬──────────────────────────────────────┐
│  💬 Chat (1/3)      │  📄 Response (2/3)                   │
│  ─────────────────  │  ─────────────────────────────────   │
│  Stats: Endpoints   │  Full markdown response displayed    │
│         Categories  │  here — wide reading area            │
│         Messages    │                                       │
│                     │                                       │
│  Example queries    │                                       │
│  ↑ clickable        │                                       │
│                     │                                       │
│  Chat history       │                                       │
│  (80-char preview)  │                                       │
│                     │                                       │
│  [Clear Chat]       │                                       │
│  [Chat input box]   │                                       │
└─────────────────────┴──────────────────────────────────────┘
```
- Left panel stores history, right panel shows full response
- UI memory only — chat history is NOT passed to Gemini queries

### v1 (Standard Chat) — `ghl_chatbot_streamlit-1.py`
```
Sidebar: Quick stats + example queries + clear button
Main: Standard st.chat_message() bubbles (full response inline)
```

---

## Repo Evolution Timeline

| Date | Commit | What Changed |
|------|--------|-------------|
| May 2025 | `first scrape w/ crawl4ai` | Basic crawler prototype |
| May 2025 | `url discovery done!` | Sitemap parsing added |
| Dec 2025 | `GHL RAG bot w Google Managed RAG v1` | Full RAG pipeline (CLI chatbot) |
| Dec 2025 | `Streamlit based GHL API Doc v2 RAG bot` | Streamlit UI added |

**Key insight:** 7-month gap between crawling and RAG. The crawler became infrastructure; the RAG chatbot was the product.

---

## Dependencies

```toml
[tool.poetry.dependencies]
python = ">=3.12,<4.0"
crawl4ai = "*"
playwright = "*"
python-dotenv = "*"
beautifulsoup4 = "*"
requests = "*"
google-genai = "^1.55.0"   # File Search API
streamlit = "^1.52.2"
```

Notable: No LangChain in the final chatbot (only in the earlier crawler pipeline). The RAG layer is entirely Google-native.

---

## Auth / Config

```
GEMINI_API_KEY    ← single env var (API key mode, not Vertex AI)
ghl_store_name.txt ← persisted store resource path
```

No GOOGLE_GENAI_USE_VERTEXAI toggle used here — this is personal/dev mode throughout (API key only).
