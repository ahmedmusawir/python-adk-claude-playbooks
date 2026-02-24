# Patterns: crawl4ai-exp-project-v1

**Project:** Crawl4AI RAG Pipeline
**Extracted:** 2026-02-23

---

## crawl4ai Patterns

### Basic AsyncWebCrawler Usage (v1 API)

```python
from crawl4ai import AsyncWebCrawler, BrowserConfig, CrawlerRunConfig, CacheMode

# Browser config — set once, reuse across all crawls
browser_config = BrowserConfig(
    headless=True,
    user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) ..."
)

# Per-run config — controls caching, timeouts
run_config = CrawlerRunConfig(
    cache_mode=CacheMode.BYPASS,  # always fresh
    page_timeout=60000,           # 60s timeout
)

async with AsyncWebCrawler(config=browser_config) as crawler:
    result = await crawler.arun(url=url, config=run_config)
```

**Pattern:** `BrowserConfig` at session level, `CrawlerRunConfig` at run level.

---

### Markdown Extraction (fit_markdown preferred)

```python
# crawl4ai returns result.markdown object (not a string directly)
# fit_markdown = cleaned/filtered markdown (preferred)
# raw_markdown = full raw markdown from page

fit_md = getattr(result.markdown, "fit_markdown", "")
raw_md = getattr(result.markdown, "raw_markdown", "")

# Prefer fit, fallback to raw
if fit_md:
    content = fit_md.strip()
elif raw_md:
    content = raw_md.strip()
else:
    content = ""  # empty page
```

**Pattern:**
- `result.markdown` is an object, not a string
- `fit_markdown` is cleaner (strips nav, footer noise)
- Always handle both being empty (page may return nothing)
- Use `getattr(..., "")` for safe attribute access

---

### Result Safety Checks

```python
result = await crawler.arun(url=url, config=run_config)

if not result:
    return ""  # no result object

if not result.success:
    # log result.status_code, result.error_message
    return ""

if result.markdown is None:
    return ""
```

**Pattern:** Three-level check — result exists, result.success, result.markdown not None.

---

### Parallel Crawl with asyncio.gather

```python
async with AsyncWebCrawler(config=browser_config) as crawler:
    tasks = [process_file(crawler, url) for url in urls]
    results = await asyncio.gather(*tasks, return_exceptions=True)

# Handle mixed results (Path | None | Exception)
for item in results:
    if isinstance(item, Path):
        created_files.append(item)
    elif isinstance(item, Exception):
        print(f"Task error: {item}")
    # None = empty page, already logged
```

**Pattern:** Share one crawler instance across all concurrent tasks. Use `return_exceptions=True` on gather.

---

### URL → Filename Slugification

```python
def slugify(text: str) -> str:
    return re.sub(r"[^a-z0-9]+", "-", text.strip().lower()).strip("-") or "root"

# Usage
name = slugify(url.replace("https://", "").replace("http://", ""))
# "https://example.com/services/" → "example-com-services"
out_path = PAGE_DIR / f"{name}.md"
```

**Pattern:** Strip scheme, slugify everything else. The `or "root"` handles edge cases.

---

## Sitemap Parsing Pattern

```python
import requests
import xml.etree.ElementTree as ET
from urllib.parse import urljoin

def fetch_sitemap_urls(base_url):
    sitemap_url = urljoin(base_url, "/sitemap.xml")

    response = requests.get(sitemap_url, timeout=10)
    root = ET.fromstring(response.content)
    namespace = {'ns': 'http://www.sitemaps.org/schemas/sitemap/0.9'}

    # Detect sitemap index (nested sitemaps)
    sitemap_tags = root.findall(".//ns:sitemap/ns:loc", namespaces=namespace)
    if sitemap_tags:
        # Sitemap index — recurse into child sitemaps
        urls = []
        for tag in sitemap_tags:
            child_urls = fetch_child_sitemap(tag.text, namespace)
            urls.extend(child_urls)
        return urls

    # Flat sitemap
    return [elem.text for elem in root.findall(".//ns:loc", namespaces=namespace)]
```

**Pattern:**
- Always use XML namespace `http://www.sitemaps.org/schemas/sitemap/0.9`
- Check for sitemap index first (`.//ns:sitemap/ns:loc`)
- Fall through to flat sitemap (`.//ns:loc`)
- Graceful error handling — return `[]` on failure

---

## LangChain Multi-Provider LLM Factory Pattern

### The Factory (`utils/global_llm.py`)

```python
def get_openai_llm(model: str, temperature: float = 0.2, **kwargs):
    try:
        from langchain_openai import ChatOpenAI
        key = kwargs.get("api_key", os.getenv("OPENAI_API_KEY"))
        if not key:
            return None
        return ChatOpenAI(model=model, api_key=key, temperature=temperature)
    except Exception as e:
        print(f"[WARN] Failed to load OpenAI LLM: {e}")
        return None
```

**Pattern per provider:**
- Lazy import (inside function — only loads if called)
- Key from kwargs first, then env var (allows override)
- Returns `None` on failure (never raises)
- Same signature across all providers

### Provider Selection (in consuming code)

```python
# Priority order: check env vars, first found wins
if OPENAI_API_KEY:
    llm = ChatOpenAI(...)
elif ANTHROPIC_API_KEY:
    llm = ChatAnthropic(...)
elif GOOGLE_API_KEY:
    llm = ChatGoogleGenerativeAI(...)
elif GROQ_API_KEY:
    llm = ChatGroq(...)
else:
    llm = None
```

**Pattern:** Cascading env var check. First available key = active provider.

### Noop Fallback

```python
def _noop(text: str) -> str:
    return text

if llm:
    def summarise(text: str) -> str:
        return llm.invoke(prompt).content.strip()
else:
    summarise = _noop  # pipeline continues without LLM
```

**Pattern:** Function assignment for no-op fallback. Pipeline degrades gracefully.

---

## Pydantic v2 Patterns

### Model Definition

```python
from pydantic import BaseModel, HttpUrl, Field
from typing import List, Optional

class Section(BaseModel):
    id: str = Field(..., description="slug identifier")
    heading: Optional[str] = Field(None)
    body: str = Field(...)
    images: List[ImageRef] = Field(default_factory=list)
```

**Pattern:**
- Use `Field(...)` for required fields
- Use `Field(None)` for optional with None default
- Use `default_factory=list` for mutable defaults (not `default=[]`)

### Serialization

```python
# Write to file (Pydantic v2)
out_path.write_text(page.model_dump_json(indent=2), encoding="utf-8")

# Read back
page = PageContent.model_validate_json(json_string)

# Dict
data = page.model_dump(mode="json", by_alias=True)
```

**Pattern:** `model_dump_json()` not `json()` (v2 API). `model_validate()` not `parse_obj()`.

---

## RAG Ingestion Pattern

### Full Pipeline (LangChain + Chroma)

```python
from langchain_community.document_loaders import UnstructuredMarkdownLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain_chroma import Chroma

# 1. Load documents
loader = UnstructuredMarkdownLoader(path)
docs = loader.load()

# 2. Chunk
splitter = RecursiveCharacterTextSplitter(chunk_size=2000, chunk_overlap=200)
chunks = splitter.split_documents(docs)

# 3. Embed + store
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vector_store = Chroma(
    collection_name=collection_name,
    embedding_function=embeddings,
    persist_directory=persist_directory,
)
vector_store.add_documents(chunks)
```

**Pattern:**
- `UnstructuredMarkdownLoader` for `.md` files
- `RecursiveCharacterTextSplitter` — recursive splitting respects document structure
- `chunk_size=2000, chunk_overlap=200` — good default for RAG
- `text-embedding-3-small` — cost-effective OpenAI embedding model
- Chroma persisted locally in `_vector-dbs/`

### Dynamic Naming from Site Config

```python
# site_config.json drives all naming — nothing hardcoded
site_key = config["site_key"]  # e.g., "cyberizegroup-com"
persist_directory = f"_vector-dbs/{site_key}-vdb"
collection_name = site_key.replace("-", "_") + "_collection"
```

**Pattern:** One `site_config.json` per project drives all downstream naming. Makes multi-site deployment trivial.

### Retrieval (MMR)

```python
retriever = vector_store.as_retriever(
    search_type="mmr",        # Maximal Marginal Relevance (diversity)
    search_kwargs={"k": 5}    # return 5 results
)

results = retriever.get_relevant_documents(query)
```

**Pattern:** MMR over pure similarity — avoids returning 5 nearly identical chunks.

---

## Section Splitting Pattern

```python
def section_splitter(markdown: str) -> list[Section]:
    lines = markdown.splitlines()
    sections = []
    current = Section(id="intro", heading=None, body="", images=[])

    for line in lines:
        if re.match(r"^#{1,3}\s+", line):  # H1, H2, H3
            if current.body.strip():       # flush previous
                sections.append(current)
            heading = line.lstrip("# ").strip()
            current = Section(id=slugify(heading), heading=heading, body="")
        else:
            current.body += line + "\n"

    if current.body.strip():               # final flush
        sections.append(current)

    return sections
```

**Pattern:**
- Stream through lines
- Trigger new section on H1-H3 (`#{1,3}`)
- Flush + create on trigger
- Final flush after loop
- First section is always `intro` (content before first heading)

---

## Terminal Output Pattern (Rich)

```python
from rich import print

# Colored status messages
print(f"[green]✔[/green] Saved: {path}")
print(f"[yellow]⚠️ Empty file:[/yellow] {filename}")
print(f"[red]❌ Failed:[/red] {error}")
print(f"[bold cyan]Processing...[/bold cyan]")
print(f"[bold]Target URLs:[/bold]")
```

**Pattern:** Rich markup tags inline in f-strings. Consistent color convention:
- `[green]` = success
- `[yellow]` = warning/skip
- `[red]` = error
- `[cyan]` = progress/info
- `[bold]` = headers

---

## File Pipeline Pattern

```python
# Stage output files consumed as next stage input
URL_FILE = OUTPUT_DIR / "discovered_pages_final.json"   # Stage 1 → Stage 2
PAGE_DIR = OUTPUT_DIR / "pages"                          # Stage 2 → Stage 3/4
PROCESSED_DIR = OUTPUT_DIR / "processed"                 # Stage 3 output
VECTOR_DIR = BASE_DIR / "_vector-dbs" / f"{site_key}-vdb"  # Stage 4 output
```

**Pattern:** Each stage knows its input path and output path. No shared state, no DB.

---

## Key Patterns Summary

| Pattern | Purpose | Location |
|---------|---------|----------|
| `BrowserConfig` + `CrawlerRunConfig` | crawl4ai session vs run config | `smart_crawler/crawler.py` |
| `fit_markdown` preference | clean vs raw output | `smart_crawler/crawler.py` |
| `result.success` check | safe crawl result handling | `smart_crawler/crawler.py` |
| `return_exceptions=True` on gather | safe concurrent crawl | `smart_crawler/crawler.py` |
| Pydantic v2 `model_dump_json()` | safe serialization | `smart_crawler/data_processing.py` |
| `_noop` fallback | LLM-optional pipeline | `smart_crawler/data_processing.py` |
| LLM factory with lazy imports | multi-provider, fail-safe | `utils/global_llm.py` |
| `site_config.json` for naming | dynamic per-site config | `rag_pipeline/vectorize_all.py` |
| `chunk_size=2000, overlap=200` | RAG chunking defaults | `rag_pipeline/vectorize_all.py` |
| MMR retrieval | diverse results | `rag_pipeline/vectorize_all.py` |
| Slugified URL → filename | deterministic file naming | `smart_crawler/crawler.py` |
| XML namespace in sitemap parse | handles all sitemap formats | `discover_site/sitemap_utils.py` |

---

_Extracted: 2026-02-23_
