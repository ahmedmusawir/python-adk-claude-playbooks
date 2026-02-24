# RAG & LLM Integration Guide: crawl4ai-exp-project-v1

**Project:** Crawl4AI RAG Pipeline
**Extracted:** 2026-02-23

---

## Overview

This document covers the RAG pipeline implementation and multi-provider LLM integration patterns used in this project. These are core patterns for any AI data ingestion system.

---

## The Full RAG Pipeline

```
Web Pages (URLs)
    ↓ crawl4ai — scrape + extract clean markdown
Markdown Files (outputs/pages/*.md)
    ↓ UnstructuredMarkdownLoader — load into LangChain Documents
    ↓ RecursiveCharacterTextSplitter — chunk
    ↓ OpenAIEmbeddings — vectorize
Chroma Vector Store (_vector-dbs/{site_key}-vdb)
    ↓ as_retriever(search_type="mmr")
Relevant Chunks (for LLM context)
```

---

## Stage 1: Content Extraction (crawl4ai)

### Installation Requirements

```bash
pip install crawl4ai playwright
crawl4ai-setup  # Installs Playwright browsers (REQUIRED before first run)
```

**Critical:** `crawl4ai-setup` must run once to install browsers. Without it, `AsyncWebCrawler` fails silently or throws obscure errors.

### Core crawl4ai Usage

```python
from crawl4ai import AsyncWebCrawler, BrowserConfig, CrawlerRunConfig, CacheMode

browser_config = BrowserConfig(
    headless=True,
    user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) ..."
)

run_config = CrawlerRunConfig(
    cache_mode=CacheMode.BYPASS,  # Skip cache for fresh content
    page_timeout=60000,           # 60 second timeout
)

async with AsyncWebCrawler(config=browser_config) as crawler:
    result = await crawler.arun(url=url, config=run_config)
```

### Accessing Results

```python
# result.markdown is an OBJECT (not string)
fit_md = getattr(result.markdown, "fit_markdown", "")   # preferred
raw_md = getattr(result.markdown, "raw_markdown", "")   # fallback

# result attributes
result.success          # bool — did the crawl succeed?
result.status_code      # HTTP status code
result.error_message    # Error description if failed
result.html             # Raw HTML (available for debugging)
```

### Cache Modes

```python
CacheMode.BYPASS    # Always fetch fresh (for production use)
CacheMode.ENABLED   # Use cache if available (for development)
CacheMode.DISABLED  # Don't cache at all
```

### fit_markdown vs raw_markdown

| | fit_markdown | raw_markdown |
|--|--|--|
| Content | Filtered clean text | Full raw markdown |
| Noise | Removes nav, footer, ads | Includes everything |
| Size | Smaller | Larger |
| Use case | RAG ingestion | Debugging |

**Always prefer `fit_markdown` for RAG pipelines.**

---

## Stage 2: Document Loading & Chunking

### Loading Markdown Files

```python
from langchain_community.document_loaders import UnstructuredMarkdownLoader

loader = UnstructuredMarkdownLoader("outputs/pages/example-com.md")
docs = loader.load()
# Returns: List[Document] — each Document has .page_content and .metadata
```

**Note:** `UnstructuredMarkdownLoader` requires the `unstructured` package.

### Chunking Strategy

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=2000,    # max tokens per chunk
    chunk_overlap=200   # overlap between chunks (preserves context)
)

chunks = splitter.split_documents(docs)
```

**`RecursiveCharacterTextSplitter` split order:**
1. `\n\n` — paragraph boundaries (preferred)
2. `\n` — line boundaries
3. ` ` — word boundaries
4. `` — character boundaries (last resort)

**Pattern:** Respects document structure before character-level splitting.

**Defaults: `chunk_size=2000, chunk_overlap=200`**
- 2000 chars ≈ ~500 tokens (leaves room in context window)
- 200 chars overlap ≈ 1-2 sentences (prevents context loss at boundaries)
- Adjust based on embedding model's token limit

---

## Stage 3: Embedding & Vector Storage

### OpenAI Embeddings

```python
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
```

**Available models:**
- `text-embedding-3-small` — 1536 dims, $0.00002/1k tokens (default)
- `text-embedding-3-large` — 3072 dims, $0.00013/1k tokens (higher quality)
- `text-embedding-ada-002` — 1536 dims (legacy, don't use for new projects)

### Chroma Vector Store

```python
from langchain_chroma import Chroma

# Create + persist
vector_store = Chroma(
    collection_name="mysite_com_collection",
    embedding_function=embeddings,
    persist_directory="_vector-dbs/mysite-com-vdb",
)
vector_store.add_documents(chunks)

# Load existing
vector_store = Chroma(
    collection_name="mysite_com_collection",
    embedding_function=embeddings,
    persist_directory="_vector-dbs/mysite-com-vdb",
)
```

**Important:** `persist_directory` must be consistent between write and read sessions.

### Naming Convention

```python
site_key = "cyberizegroup-com"  # from site_config.json
persist_directory = f"_vector-dbs/{site_key}-vdb"
collection_name = site_key.replace("-", "_") + "_collection"
# → "_vector-dbs/cyberizegroup-com-vdb"
# → "cyberizegroup_com_collection"
```

**Rule:** Collection names can't contain hyphens (Chroma restriction) — replace `-` with `_`.

---

## Stage 4: Retrieval

### Retriever Setup

```python
# Simple similarity retrieval
retriever = vector_store.as_retriever(search_kwargs={"k": 5})

# MMR retrieval (preferred for diverse results)
retriever = vector_store.as_retriever(
    search_type="mmr",
    search_kwargs={"k": 5}
)
```

### Query

```python
results = retriever.get_relevant_documents(query)
# Returns: List[Document]
# Each Document: .page_content (chunk text), .metadata (source file, etc.)
```

### Search Types

| Type | Method | Best For |
|------|--------|----------|
| `similarity` | Cosine similarity | Precise fact retrieval |
| `mmr` | Maximal Marginal Relevance | Synthesis, broad coverage |
| `similarity_score_threshold` | Similarity with min threshold | Filter low-quality results |

---

## Multi-Provider LLM Factory

### Factory Functions (`utils/global_llm.py`)

```python
from utils.global_llm import (
    get_openai_llm,
    get_anthropic_llm,
    get_google_llm,
    get_groq_llm,
    get_ollama_llm,
)

# All return LangChain chat model or None
llm = get_openai_llm("gpt-4o", temperature=0.2)
llm = get_anthropic_llm("claude-sonnet-4-6", temperature=0.2)
llm = get_google_llm("gemini-2.0-flash", temperature=0.2)
llm = get_groq_llm("llama3-8b-8192", temperature=0.2)
llm = get_ollama_llm("deepseek-r1:1.5b")  # local model, no key needed
```

### All Return the Same Interface

```python
# Works identically regardless of provider
response = llm.invoke([HumanMessage(content="Your prompt here")])
text = response.content
```

### Environment Variable Mapping

```
OPENAI_API_KEY     → ChatOpenAI
ANTHROPIC_API_KEY  → ChatAnthropic
GOOGLE_API_KEY     → ChatGoogleGenerativeAI
GROQ_API_KEY       → ChatGroq
(none needed)      → ChatOllama (local)
```

### Ollama (Local Models)

```python
# No API key needed — runs locally
llm = get_ollama_llm("deepseek-r1:1.5b")
# or with custom base URL
llm = get_ollama_llm("llama3", base_url="http://localhost:11434")
```

**Use case:** Development without API costs. Good for testing pipeline logic.

---

## RAG Cost Estimation

### Per Site Ingestion (OpenAI)

| Step | Cost | Notes |
|------|------|-------|
| Embeddings (100 pages, ~500 tokens each) | ~$0.50 | text-embedding-3-small |
| Summarization (optional, 100 sections) | ~$0.20-0.50 | gpt-4o-mini |
| **Total ingestion** | **~$0.70-1.00** | One-time per site |

### Per Query

| Step | Cost | Notes |
|------|------|-------|
| Retrieval (k=5 chunks) | $0.00 | Already embedded |
| LLM response (if used) | ~$0.01-0.05 | Depends on model |

**Key insight:** Embedding cost is one-time (at ingestion). Query cost is just LLM tokens.

---

## Common Issues & Solutions

### crawl4ai: Empty Markdown

**Problem:** `fit_markdown` returns empty string for some pages.
**Causes:**
- Page requires JavaScript to render (JS-heavy SPAs)
- Page blocked by bot detection
- Page has `noindex` / behind login

**Solutions:**
```python
# Add delay to let JS render
run_config = CrawlerRunConfig(
    cache_mode=CacheMode.BYPASS,
    page_timeout=60000,
    # Try these if empty:
    # wait_for_network_idle=True,
    # delay_before_return_html=2.0,
)
```

### Chroma: Collection Name Restriction

**Problem:** `ValueError: Collection name must only contain characters a-z, A-Z, 0-9, _, or -`
**Actually:** Hyphens ARE allowed in Chroma names. But underscores are safer.
**Pattern:** Always convert `site_key` hyphens to underscores for collection names.

### LangChain: Version Conflicts

**Problem:** `langchain` has frequent breaking changes between minor versions.
**Solution:** Pin versions in `pyproject.toml`. Check deprecation warnings and update gradually.

### Unstructured: Missing Dependencies

**Problem:** `UnstructuredMarkdownLoader` requires additional system packages.
**Solution:**
```bash
pip install unstructured
# May also need:
pip install unstructured[md]
```

---

## Key Learnings for Manuals

1. **`crawl4ai-setup` is required** — must run once before any crawling
2. **`fit_markdown` over `raw_markdown`** — always for RAG pipelines
3. **`result.markdown` is an object** — access `.fit_markdown`, don't use as string
4. **chunk_size=2000, overlap=200** — good RAG defaults
5. **MMR over similarity** — better diversity for website content
6. **LLM factory pattern** — multi-provider support from the start
7. **Optional LLM** — always design for LLM-optional (graceful degradation)
8. **Chroma collection names** — no hyphens, use underscores
9. **`_vector-dbs/` must be gitignored** — large binary files
10. **One-time embedding cost** — ingestion is expensive, querying is cheap

---

_Extracted: 2026-02-23_
