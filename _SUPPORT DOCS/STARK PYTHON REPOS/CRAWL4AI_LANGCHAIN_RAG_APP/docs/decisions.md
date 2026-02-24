# Key Decisions: crawl4ai-exp-project-v1

**Project:** Crawl4AI RAG Pipeline
**Extracted:** 2026-02-23

---

## Decision 1: crawl4ai over Direct Playwright/Selenium

**Context:**
Need to scrape multiple web pages and extract clean content.

**Decision:**
Use `crawl4ai` library (wraps Playwright) rather than raw Playwright or Scrapy.

**Alternatives Considered:**
1. Raw Playwright — direct browser control
2. Scrapy — full crawl framework
3. BeautifulSoup + requests — lightweight HTML parsing
4. crawl4ai — AI-first web crawler

**Rationale:**
- `crawl4ai` returns `fit_markdown` — pre-cleaned content, strips nav/footer/ads automatically
- Designed for LLM/RAG use cases (clean text extraction is the whole point)
- Async-native — same async model as the rest of the pipeline
- Headless browser handles JS-rendered content (SPA sites)

**Trade-offs:**
- ✅ Clean markdown output without manual parsing
- ✅ Handles JS-heavy sites
- ✅ Async-native
- ❌ Playwright dependency (browser install required: `crawl4ai-setup`)
- ❌ Heavier than requests + BeautifulSoup for simple sites
- ❌ Newer library — API changes between versions

**Outcome:**
Right tool for RAG content extraction. `fit_markdown` alone justifies the choice.

---

## Decision 2: LangChain as LLM Abstraction Layer

**Context:**
Need LLM calls for optional content summarization and future prompt generation.

**Decision:**
Use LangChain (`langchain`, `langchain-openai`, `langchain-anthropic`, etc.) rather than calling provider SDKs directly.

**Alternatives Considered:**
1. Direct OpenAI SDK (`openai.ChatCompletion`)
2. Direct Anthropic SDK (`anthropic.Anthropic`)
3. LangChain — provider abstraction
4. LiteLLM — lightweight multi-provider proxy

**Rationale:**
- Multi-provider support from day one (future-proof)
- LangChain ecosystem: document loaders, text splitters, retrievers all use same Document type
- `langchain_chroma`, `langchain_openai`, `langchain_community` work seamlessly together
- Easy swap: change provider by swapping one class

**Trade-offs:**
- ✅ Provider-agnostic code
- ✅ Rich ecosystem (loaders, splitters, retrievers)
- ✅ Consistent Document type through pipeline
- ❌ LangChain complexity (many packages, frequent version changes)
- ❌ Abstraction overhead (harder to debug underlying API calls)
- ❌ Heavy dependency footprint

**Outcome:**
Good choice for RAG pipeline — LangChain's document pipeline is designed for this exact use case.

---

## Decision 3: Chroma as Local Vector Store (Not Cloud)

**Context:**
Need a vector database to store and query embedded content chunks.

**Decision:**
Use Chroma persisted locally (`_vector-dbs/`) rather than a cloud vector DB.

**Alternatives Considered:**
1. Pinecone — managed cloud vector DB
2. Weaviate — self-hosted or cloud
3. pgvector — PostgreSQL extension
4. FAISS — in-memory, no persistence
5. Chroma — local persistent vector DB

**Rationale:**
- Zero infrastructure — no API keys, no billing, runs locally
- Persistent — `persist_directory` survives process restarts
- Simple API — `Chroma(collection_name=..., embedding_function=..., persist_directory=...)`
- LangChain native integration (`langchain-chroma`)
- Right scale for single-site RAG

**Trade-offs:**
- ✅ No cloud setup
- ✅ Persistent locally
- ✅ Fast for small-medium collections
- ❌ Not scalable to millions of vectors
- ❌ Single-machine only (can't share across services)
- ❌ `_vector-dbs/` can get large — must gitignore

**Outcome:**
Perfect for per-site local RAG. Gitignore `_vector-dbs/` to keep repo clean.

---

## Decision 4: OpenAI Embeddings (text-embedding-3-small)

**Context:**
Need an embedding model to vectorize text chunks.

**Decision:**
Use `OpenAIEmbeddings(model="text-embedding-3-small")` as default.

**Alternatives Considered:**
1. `text-embedding-ada-002` — older OpenAI model
2. `text-embedding-3-small` — newer, cheaper, better
3. `text-embedding-3-large` — highest quality
4. Google Vertex embeddings
5. Local embeddings (sentence-transformers via Ollama)

**Rationale:**
- `text-embedding-3-small` is newer and better than ada-002 at lower cost
- Fast, reliable, 1536-dimensional vectors
- Works with Chroma out of the box
- Good quality/cost balance for RAG use case

**Trade-offs:**
- ✅ Excellent quality for RAG retrieval
- ✅ Cost-effective (~$0.00002/1k tokens)
- ✅ Battle-tested
- ❌ Requires OpenAI API key
- ❌ Embedding model lock-in (changing models = re-embed all docs)

**Outcome:**
Industry standard default. Could switch to local embeddings (Ollama) for zero-cost option later.

---

## Decision 5: site_config.json for Dynamic Naming

**Context:**
Need to handle multiple sites without hardcoded directory names.

**Decision:**
Drive all naming from `site_config.json` with a `site_key` field.

**Alternatives Considered:**
1. Hardcode `site_key` in script
2. CLI argument
3. Environment variable
4. Config file (`site_config.json`)

**Rationale:**
- Config file is portable (travels with project)
- Easily editable before a run
- Drives both `persist_directory` AND `collection_name` from one value
- Can be generated by earlier pipeline stage in future

**Structure:**
```json
{"site_key": "cyberizegroup-com"}
```

**Naming convention:**
- `site_key` = domain slugified: `cyberizegroup.com` → `cyberizegroup-com`
- `persist_directory`: `_vector-dbs/{site_key}-vdb`
- `collection_name`: `{site_key}_collection` (dashes → underscores)

**Trade-offs:**
- ✅ Clean multi-site support
- ✅ Self-documenting naming
- ✅ Easy to automate
- ❌ Extra file to manage
- ❌ Easy to forget to update before running

**Outcome:**
Good pattern for multi-site pipelines. Consistent naming makes debugging easy.

---

## Decision 6: File-Based Pipeline (No Database)

**Context:**
Need state management across 4-5 pipeline stages.

**Decision:**
Each stage reads/writes files. No shared database or in-memory state.

**Rationale:**
- Same reasoning as VidGen: simplicity, debuggability, portability
- Can inspect every stage's output by opening files
- Easy to re-run from any stage (delete output files of that stage, re-run)
- Files are the "handshake" between stages

**Trade-offs:**
- ✅ Zero infrastructure
- ✅ Human-readable state
- ✅ Resume from any stage
- ❌ No query capabilities
- ❌ File management required

**Outcome:**
Proven pattern (same as VidGen). Correct choice for pipeline tools.

---

## Decision 7: LLM Optional in Processing Stage

**Context:**
Section summarization in `data_processing.py` requires an LLM, but the pipeline should work without one.

**Decision:**
LLM is optional — if no API key, summarize with `_noop` (return text unchanged).

**Rationale:**
- Pipeline has value without summarization (structured JSON is useful)
- Not everyone has API keys at all times
- Development/testing can run without spending tokens
- Graceful degradation > hard failure

**Implementation:**
```python
def _noop(text: str) -> str:
    return text

summarise = actual_summarise if llm else _noop
```

**Outcome:**
Good resilience pattern. Applies broadly to any optional AI enhancement.

---

## Decision 8: MMR Retrieval (Not Pure Similarity)

**Context:**
Need retrieval strategy for the Chroma vector store.

**Decision:**
Use MMR (Maximal Marginal Relevance) instead of pure similarity search.

**Alternatives Considered:**
1. Pure similarity — most similar chunks
2. MMR — balance similarity + diversity

**Rationale:**
- Website content often has repetition (same info on multiple pages)
- Pure similarity would return 5 nearly-identical chunks
- MMR ensures diverse results (covers more of the site)
- Better for synthesis tasks (LLM gets a broader picture)

**Trade-offs:**
- ✅ More diverse results
- ✅ Better for content synthesis
- ❌ Slightly slower than pure similarity
- ❌ `k=5` may include less-relevant results

**Outcome:**
Right default for website content RAG. Pure similarity fine for focused fact retrieval.

---

## Decision 9: Poetry for Dependency Management

**Context:**
Need Python package management for multi-package project.

**Decision:**
Use Poetry (`pyproject.toml`) rather than pip + requirements.txt.

**Rationale:**
- Lockfile (`poetry.lock`) — reproducible environments
- Package includes (`packages = [...]`) — each module is importable
- Dev dependencies separate from runtime
- Standard modern Python practice

**Outcome:**
No issues. Poetry handles the multi-package structure cleanly.

---

## Key Takeaways

**Decision Priorities Revealed:**

1. **AI-native tools > general tools** (crawl4ai over Scrapy/requests)
2. **Abstraction > direct API** (LangChain over OpenAI SDK)
3. **Local > cloud** (Chroma locally > Pinecone)
4. **Optional AI > required AI** (LLM optional in processing)
5. **File state > database** (same as VidGen — consistent pattern)
6. **Dynamic config > hardcoded** (`site_config.json` drives naming)
7. **Quality + diversity > raw similarity** (MMR retrieval)

---

_Extracted: 2026-02-23_
