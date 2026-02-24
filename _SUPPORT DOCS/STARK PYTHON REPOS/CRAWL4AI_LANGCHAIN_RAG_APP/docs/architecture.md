# Architecture: crawl4ai-exp-project-v1

**Project:** Crawl4AI RAG Pipeline — Website-to-Vector-Store
**Extracted:** 2026-02-23
**Branch:** rag-layer

---

## Overview

This project is a **website-to-RAG pipeline**. It ingests a website (via sitemap discovery + crawl4ai scraping), processes the content into structured form, and stores it in a local Chroma vector database — ready for LLM-powered retrieval.

**End goal:** Feed crawled site content into a Lovable prompt agent to generate UI/app builds.

**Key characteristic:** Fully sequential pipeline, each stage outputs files consumed by the next.

---

## High-Level Flow

```
[Website]
    ↓
Stage 1: Discover (sitemap_utils.py)
    → outputs/discovered_pages_final.json  (list of URLs)
    ↓
Stage 2: Crawl (smart_crawler/crawler.py)
    → outputs/pages/*.md  (one markdown file per page)
    ↓
Stage 3: Process (smart_crawler/data_processing.py)
    → outputs/processed/*.json  (Pydantic PageContent → structured JSON)
    ↓
Stage 4: Vectorize (rag_pipeline/vectorize_all.py)
    → _vector-dbs/{site_key}-vdb/  (Chroma local vector store)
    ↓
Stage 5: Query / Prompt Agent
    → Interactive retrieval OR Lovable prompt generation
```

---

## File Structure

```
crawl4ai-exp-project-v1/
├── discover_site/
│   └── sitemap_utils.py          # Sitemap XML → URL list
├── smart_crawler/
│   ├── crawler.py                # crawl4ai → markdown files
│   ├── data_processing.py        # Markdown → structured JSON (LLM optional)
│   ├── schema.py                 # Pydantic models (PageContent, Section)
│   └── utils.py                  # (stub)
├── rag_pipeline/
│   ├── vectorize_all.py          # Chunk + embed + store in Chroma
│   └── __init__.py
├── prompt_agent/
│   ├── lovable_prompter.py       # (stub) Reads JSON → generates Lovable prompts
│   └── utils.py                  # (stub)
├── utils/
│   └── global_llm.py             # Multi-provider LangChain LLM factory
├── outputs/
│   ├── discovered_pages_final.json   # Stage 1 output
│   ├── pages/*.md                    # Stage 2 output
│   ├── processed/*.json              # Stage 3 output
│   ├── homepage.json                 # (legacy structured output)
│   └── services.json                 # (legacy structured output)
├── final_prompts/
│   └── lovable_full_prompt.txt   # Final assembled prompt for Lovable
├── _vector-dbs/                  # Chroma vector DBs (gitignored)
│   └── {site_key}-vdb/
├── _old_keep/
│   └── main.py                   # Original prototype (basic crawl4ai usage)
├── scripts/
│   └── init-folders.sh           # Scaffold script for project structure
├── site_config.json              # Per-site config (site_key drives naming)
├── pyproject.toml                # Poetry dependencies
└── .env                          # API keys (gitignored)
```

---

## Stage Details

### Stage 1: Site Discovery (`discover_site/sitemap_utils.py`)

**Input:** Website base URL
**Output:** `outputs/discovered_pages_final.json` — array of `{"url": "..."}` objects

**Logic:**
- Fetches `/sitemap.xml` from base URL
- Handles both flat sitemaps AND sitemap index files (nested sitemaps)
- Falls back gracefully if sitemap fetch/parse fails
- Returns list of URLs

**Output format:**
```json
[
  {"url": "https://example.com"},
  {"url": "https://example.com/services/"},
  ...
]
```

---

### Stage 2: Crawl (`smart_crawler/crawler.py`)

**Input:** `outputs/discovered_pages_final.json`
**Output:** `outputs/pages/{slugified-url}.md` — one markdown file per page

**Key behaviors:**
- Uses `crawl4ai.AsyncWebCrawler` with Playwright headless browser
- Prefers `result.markdown.fit_markdown` (clean), falls back to `raw_markdown`
- Skips pages with empty content (warns, doesn't error)
- Slugifies URL to create filename: `https://example.com/services/` → `example-com-services.md`
- User confirmation prompt before starting crawl
- `CacheMode.BYPASS` — always fresh, no caching
- 60-second page timeout

---

### Stage 3: Process (`smart_crawler/data_processing.py`)

**Input:** `outputs/pages/*.md`
**Output:** `outputs/processed/{filename}.json` — Pydantic `PageContent` as JSON

**Logic:**
- Reads each markdown file
- Splits into `Section` objects on H1-H3 headings
- Optionally summarizes each section via LLM (multi-provider, env-key driven)
- Serializes to JSON via Pydantic v2 `.model_dump_json()`
- User confirmation prompt before processing

**LLM is optional:** If no API keys in `.env`, falls back to raw text (no summarization).

---

### Stage 4: Vectorize (`rag_pipeline/vectorize_all.py`)

**Input:** `outputs/pages/*.md` + `site_config.json`
**Output:** `_vector-dbs/{site_key}-vdb/` — local Chroma vector store

**Logic:**
1. Load `site_config.json` to get `site_key` (drives directory + collection naming)
2. Load all `.md` files via `UnstructuredMarkdownLoader`
3. Chunk with `RecursiveCharacterTextSplitter(chunk_size=2000, chunk_overlap=200)`
4. Embed with `OpenAIEmbeddings(model="text-embedding-3-small")`
5. Store in Chroma with `persist_directory` derived from `site_key`
6. Interactive retrieval test loop (MMR, k=5)

---

### Stage 5: Prompt Agent (`prompt_agent/lovable_prompter.py`)

**Status:** Stub (not yet implemented)
**Intent:** Read processed JSON → assemble full Lovable UI prompt → write to `final_prompts/lovable_full_prompt.txt`

---

## Data Models (`smart_crawler/schema.py`)

```
PageContent
├── page: str          (route slug, e.g. "homepage")
├── url: HttpUrl       (canonical URL)
└── sections: List[Section]
    └── Section
        ├── id: str          (slug of heading)
        ├── heading: Optional[str]
        ├── body: str        (markdown text)
        └── images: List[ImageRef]
            └── ImageRef
                ├── src: HttpUrl
                └── alt: Optional[str]
```

**Pydantic v2** — uses `.model_dump_json()`, `.model_validate()`.

---

## LLM Layer (`utils/global_llm.py`)

Multi-provider LangChain factory. Supports:
- OpenAI (`langchain_openai.ChatOpenAI`)
- Anthropic (`langchain_anthropic.ChatAnthropic`)
- Google Gemini (`langchain_google_genai.ChatGoogleGenerativeAI`)
- Groq (`langchain_groq.ChatGroq`)
- Ollama (`langchain_ollama.ChatOllama`) — local models

Provider selected based on which API key is present in `.env`. Fails silently (returns `None`) if key missing or import fails.

---

## Configuration

### `site_config.json` (per-site)
```json
{
  "site_key": "cyberizegroup-com"
}
```
Drives:
- Chroma `persist_directory`: `_vector-dbs/cyberizegroup-com-vdb`
- Chroma `collection_name`: `cyberizegroup_com_collection`

### `.env` (API keys, gitignored)
```
OPENAI_API_KEY=...
ANTHROPIC_API_KEY=...
GOOGLE_API_KEY=...
GROQ_API_KEY=...
```

### `pyproject.toml` (Poetry)
Key dependencies: `crawl4ai`, `playwright`, `langchain`, `langchain-chroma`, `pydantic ^2`, `unstructured`, `langchain-ollama`

---

## Key Design Characteristics

1. **File-based pipeline** — each stage reads/writes files, no shared state
2. **Sequential by design** — stages run in order, outputs feed next stage
3. **LLM is optional** — pipeline works without any API keys (degraded mode)
4. **Multi-provider LLM** — not locked to one vendor
5. **Local vector DB** — Chroma persisted locally (no cloud DB needed)
6. **Gitignored outputs** — `_vector-dbs/` not committed (can be large)
7. **User confirmation gates** — crawl and process stages prompt before running

---

## Evolution / History

**Original prototype** (`_old_keep/main.py`): Single-URL basic crawl4ai demo — `AsyncWebCrawler().arun(url)` → write markdown. ~35 lines.

**Current state:** Full multi-stage pipeline with discovery, smart crawl, structured processing, RAG vectorization, and stubs for prompt generation.

**Branch `rag-layer`:** Active development adding the vector store layer to what was previously a crawl + process pipeline.

---

_Extracted: 2026-02-23_
