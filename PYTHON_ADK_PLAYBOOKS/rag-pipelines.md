# RAG Pipelines

## What This Is For

Two RAG approaches: Google Managed RAG (File Search API) and DIY RAG (crawl4ai + LangChain + Chroma). Production patterns from the GHL API docs chatbot (700+ endpoints, 1415 files) and the crawl4ai scraping pipeline.

**Use Managed RAG unless you need custom chunking or retrieval.** The managed path is significantly simpler.

---

## Track A: Google Managed RAG (Recommended)

### Quick Reference — Full Lifecycle

```python
import os
os.environ["GOOGLE_GENAI_USE_VERTEXAI"] = "1"

from google import genai
from google.genai import types
import time

client = genai.Client()
MODEL = "gemini-2.5-flash"

# 1. Create store
store = client.files.create_file_search_store(display_name="my-knowledge-base")
store_name = store.name  # Save this — needed for all operations

# 2. Upload document (async — must poll)
with open("my_doc.pdf", "rb") as f:
    operation = client.files.upload_to_file_search_store(
        file_search_store_name=store_name,
        file=f,
        display_name="My Document",
        mime_type="application/pdf",
    )

# 3. Poll until done
while not operation.done:
    time.sleep(1)
    operation = client.operations.get(name=operation.name)

# 4. Query
response = client.models.generate_content(
    model=MODEL,
    contents="What does the document say about authentication?",
    config=types.GenerateContentConfig(
        tools=[types.Tool(file_search=types.FileSearchTool(file_search_stores=[store_name]))],
    ),
)
print(response.text)

# 5. Delete store when done (force=True required if store has files)
client.files.delete_file_search_store(name=store_name, force=True)
```

---

### Pattern 1: Create Store

```python
def create_store(display_name: str) -> str:
    """Create a File Search store. Returns store.name (full resource path)."""
    store = client.files.create_file_search_store(display_name=display_name)
    return store.name  # Format: "fileSearchStores/abc123"

# Persist the store name — it's the key to all operations
store_name = create_store("ghl-api-docs")
with open("store_name.txt", "w") as f:
    f.write(store_name)
```

---

### Pattern 2: Upload File with Polling

Upload is async. The operation must complete before the file is queryable.

```python
def upload_file_and_wait(
    store_name: str,
    file_path: str,
    display_name: str,
    mime_type: str = "application/pdf",
) -> bool:
    """Upload file to store, poll until done. Returns True on success."""
    try:
        with open(file_path, "rb") as f:
            operation = client.files.upload_to_file_search_store(
                file_search_store_name=store_name,
                file=f,
                display_name=display_name,
                mime_type=mime_type,
            )

        # Poll until processing complete
        max_wait = 300  # seconds
        elapsed = 0
        while not operation.done:
            time.sleep(2)
            elapsed += 2
            operation = client.operations.get(name=operation.name)
            if elapsed > max_wait:
                print(f"Upload timeout: {display_name}")
                return False

        if operation.error and operation.error.code != 0:
            print(f"Upload error for {display_name}: {operation.error.message}")
            return False

        return True

    except Exception as e:
        print(f"Upload failed for {display_name}: {e}")
        return False
```

---

### Pattern 3: Batch Upload with Per-File Error Handling

```python
def batch_upload(store_name: str, documents: list[dict]) -> dict:
    """
    Upload multiple files. documents = [{path, display_name, mime_type}].
    Returns {success: [...], failed: [...]}.
    """
    results = {"success": [], "failed": []}

    for doc in documents:
        print(f"Uploading: {doc['display_name']}")
        ok = upload_file_and_wait(
            store_name=store_name,
            file_path=doc["path"],
            display_name=doc["display_name"],
            mime_type=doc.get("mime_type", "application/pdf"),
        )
        if ok:
            results["success"].append(doc["display_name"])
        else:
            results["failed"].append(doc["display_name"])

    print(f"Upload complete: {len(results['success'])} ok, {len(results['failed'])} failed")
    return results
```

---

### Pattern 4: Dual-Document Strategy (Production Quality Enhancement)

Upload each document twice: once as the original, once as a structured summary. The summary helps the model retrieve the right context for broad queries. The original gives complete detail for specific queries.

```python
SUMMARY_PROMPT = """Analyze this document and create a structured summary with these sections:

## Document Overview
Brief description of what this document covers.

## Key Topics
Bullet list of main topics covered.

## Important Details
Key facts, numbers, names, procedures mentioned.

## Use Cases
What questions can this document answer?

Keep the summary comprehensive but structured for search retrieval."""


def create_and_upload_summary(
    store_name: str,
    original_content: str,
    doc_name: str,
    tmp_dir: str = "/tmp",
) -> bool:
    """Generate summary and upload alongside original."""
    # Generate summary
    summary_response = client.models.generate_content(
        model="gemini-2.5-flash",
        contents=f"{SUMMARY_PROMPT}\n\nDocument content:\n{original_content}"
    )
    summary_text = summary_response.text

    # Save to temp file
    summary_path = f"{tmp_dir}/{doc_name}_summary.txt"
    with open(summary_path, "w") as f:
        f.write(summary_text)

    return upload_file_and_wait(
        store_name=store_name,
        file_path=summary_path,
        display_name=f"{doc_name} - Structured Summary",
        mime_type="text/plain",
    )
```

---

### Pattern 5: System Prompt as Retrieval Strategy

Tell the model to use the File Search store. Don't assume it will do RAG automatically.

```python
SYSTEM_PROMPT = """You are a documentation assistant for the GHL API.

When answering questions:
1. ALWAYS search the File Search store for relevant documentation
2. Cite the specific endpoint or section you're referencing
3. If the documentation doesn't cover the question, say so clearly
4. For code examples, use the exact parameters and formats shown in the docs

You have access to comprehensive API documentation. Use it."""

def query_knowledge_base(store_name: str, question: str, conversation_history: list = None) -> str:
    """Query the File Search store with optional conversation history."""
    messages = list(conversation_history) if conversation_history else []
    messages.append({
        "role": "user",
        "parts": [{"text": question}]
    })

    response = client.models.generate_content(
        model="gemini-2.5-flash",
        contents=messages,
        config=types.GenerateContentConfig(
            system_instruction=SYSTEM_PROMPT,
            tools=[types.Tool(
                file_search=types.FileSearchTool(
                    file_search_stores=[store_name]
                )
            )],
        ),
    )
    return response.text
```

---

### Pattern 6: Filename-Based Category Extraction

More reliable than asking the LLM to classify documents. Extract category from the filename pattern.

```python
import re

def extract_category_from_filename(filename: str) -> str:
    """
    Extract category from structured filenames.
    Examples:
      "contacts_search_endpoint.md" → "contacts"
      "conversations_get_by_id.pdf" → "conversations"
      "GHL_API_v2_webhooks.txt" → "webhooks"
    """
    # Lowercase, replace separators
    name = filename.lower().replace("-", "_").replace(" ", "_")
    # Remove extension
    name = re.sub(r'\.[a-z]+$', '', name)
    # First segment before underscore is usually the category
    parts = name.split("_")
    return parts[0] if parts else "general"

# For the GHL API chatbot: ~700 endpoints across categories
# Filename pattern: "contacts_create.md", "conversations_list.md", etc.
# → categories: contacts, conversations, workflows, opportunities, campaigns, ...
```

---

### Pattern 7: Synthetic Master Index

One document that describes the entire knowledge base structure. Enables category-level queries ("what CRM modules are available?") without searching every document.

```python
def create_master_index(store_name: str, documents: list[dict]) -> bool:
    """Create a synthetic index document from the list of uploaded docs."""

    # Group by category
    by_category = {}
    for doc in documents:
        cat = extract_category_from_filename(doc["display_name"])
        by_category.setdefault(cat, []).append(doc["display_name"])

    # Build index content
    lines = ["# Knowledge Base Master Index\n"]
    lines.append(f"Total documents: {len(documents)}\n")
    lines.append(f"Categories: {len(by_category)}\n\n")

    for category, docs in sorted(by_category.items()):
        lines.append(f"## {category.title()} ({len(docs)} documents)\n")
        for doc in sorted(docs):
            lines.append(f"- {doc}\n")
        lines.append("\n")

    index_content = "".join(lines)
    index_path = "/tmp/master_index.md"
    with open(index_path, "w") as f:
        f.write(index_content)

    return upload_file_and_wait(
        store_name=store_name,
        file_path=index_path,
        display_name="MASTER INDEX - Knowledge Base Overview",
        mime_type="text/markdown",
    )
```

---

### Pattern 8: Delete Document with force=True

```python
def delete_document(store_name: str, document_name: str) -> bool:
    """Delete a specific document from a store."""
    try:
        # document_name format: "fileSearchStores/abc/files/xyz"
        client.files.delete_file_search_store_file(name=document_name, force=True)
        return True
    except Exception as e:
        print(f"Delete failed: {e}")
        return False

def delete_store(store_name: str) -> bool:
    """Delete entire store. force=True required when store has files."""
    try:
        client.files.delete_file_search_store(name=store_name, force=True)
        return True
    except Exception as e:
        print(f"Store delete failed: {e}")
        return False
```

**force=True gotcha:** If you try to delete a store that still contains files without `force=True`, you get a 400 error. Always use `force=True` for store deletion.

---

### Pattern 9: Store Name as Persistent State

Store the store name (a full resource path like `fileSearchStores/abc123`) in a file. Creating a new store on every startup defeats the purpose of RAG.

```python
STORE_NAME_FILE = "store_name.txt"

def get_or_create_store(display_name: str) -> str:
    """Load existing store name or create new one."""
    if os.path.exists(STORE_NAME_FILE):
        with open(STORE_NAME_FILE) as f:
            store_name = f.read().strip()
        # Verify it still exists
        try:
            client.files.get_file_search_store(name=store_name)
            return store_name
        except Exception:
            print("Store no longer exists, creating new one")

    store = client.files.create_file_search_store(display_name=display_name)
    with open(STORE_NAME_FILE, "w") as f:
        f.write(store.name)
    return store.name
```

---

### Auth: API Key vs Vertex AI

File Search API supports two auth modes:

**API Key mode (simpler):**
```python
import os
client = genai.Client(api_key=os.environ["GEMINI_API_KEY"])
```

**Vertex AI mode (production):**
```python
os.environ["GOOGLE_GENAI_USE_VERTEXAI"] = "1"
client = genai.Client()  # Uses ADC or service account
```

Use API key mode for prototyping. Use Vertex AI mode for production (better rate limits, billing control, service account auth).

---

### Managed RAG Gotchas

**1. force=True on store deletion**
`delete_file_search_store(name=..., force=True)` — without `force=True`, deletion fails if store has files.

**2. Upload is async — must poll**
The upload operation is not instant. The file is in state PROCESSING for 1-30 seconds. Query it before it's ACTIVE and you get no results.

**3. store.name is a full resource path**
`store.name` = `"fileSearchStores/abc123456"`. This is what you save and pass to all subsequent calls. Don't confuse with `store.display_name`.

**4. No bulk delete of documents**
There's no batch delete API. To clear a store, delete it with `force=True` and recreate, or delete documents one by one.

**5. LLM category extraction is unreliable**
Asking the model to categorize documents from their content produces inconsistent results. Extract categories from structured filenames — more reliable.

**6. Implicit retrieval requires instruction**
The model won't automatically use the File Search store just because it's in the config. Include explicit retrieval instructions in the system prompt.

---

## Track B: DIY RAG (crawl4ai + LangChain + Chroma)

Use this when you need full control over chunking, embedding, or retrieval strategy. More work, more control.

### crawl4ai — Web Scraping

```python
from crawl4ai import AsyncWebCrawler, CrawlerRunConfig
import asyncio

async def scrape_url(url: str) -> str:
    """Scrape a URL and return clean markdown content."""
    config = CrawlerRunConfig(
        word_count_threshold=50,        # Skip short text blocks
        remove_overlay_elements=True,   # Remove popups/banners
        exclude_external_links=True,
    )

    async with AsyncWebCrawler() as crawler:
        result = await crawler.arun(url=url, config=config)

        if not result.success:
            raise ValueError(f"Crawl failed: {result.error_message}")

        # Use fit_markdown for cleaner output (removes nav, footers)
        content = result.markdown_v2.fit_markdown or result.markdown
        return content


async def scrape_urls(urls: list[str]) -> dict[str, str]:
    """Scrape multiple URLs. Returns {url: content} dict."""
    results = {}
    async with AsyncWebCrawler() as crawler:
        for url in urls:
            try:
                config = CrawlerRunConfig(word_count_threshold=50)
                result = await crawler.arun(url=url, config=config)
                if result.success:
                    results[url] = result.markdown_v2.fit_markdown or result.markdown
                else:
                    print(f"Failed: {url} — {result.error_message}")
            except Exception as e:
                print(f"Error scraping {url}: {e}")
    return results


# Run from sync code:
contents = asyncio.run(scrape_urls(url_list))
```

### Ingest: Chunk → Embed → Store

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain_chroma import Chroma
import re
from pathlib import Path

COLLECTION_NAME = "my_knowledge_base"
PERSIST_DIR = "./chroma_db"

def slugify(url: str) -> str:
    """Convert URL to safe filename."""
    return re.sub(r'[^a-z0-9-]', '-', url.lower())[:60]

def ingest_documents(documents: dict[str, str]) -> Chroma:
    """
    Chunk, embed, and store documents.
    documents = {source_url: content_text}
    """
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=1000,
        chunk_overlap=200,
        separators=["\n\n", "\n", " ", ""],
    )

    all_chunks = []
    all_metadata = []
    all_ids = []

    for source, content in documents.items():
        chunks = splitter.split_text(content)
        for i, chunk in enumerate(chunks):
            all_chunks.append(chunk)
            all_metadata.append({"source": source, "chunk": i})
            all_ids.append(f"{slugify(source)}-{i}")

    embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

    vectorstore = Chroma(
        collection_name=COLLECTION_NAME,
        embedding_function=embeddings,
        persist_directory=PERSIST_DIR,
    )

    vectorstore.add_texts(
        texts=all_chunks,
        metadatas=all_metadata,
        ids=all_ids,
    )

    print(f"Ingested {len(all_chunks)} chunks from {len(documents)} documents")
    return vectorstore
```

### Retrieve and Generate

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain.schema import HumanMessage, SystemMessage

def load_vectorstore() -> Chroma:
    """Load existing Chroma vectorstore from disk."""
    embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
    return Chroma(
        collection_name=COLLECTION_NAME,
        embedding_function=embeddings,
        persist_directory=PERSIST_DIR,
    )

def query_rag(question: str, vectorstore: Chroma, k: int = 5) -> str:
    """Retrieve relevant chunks and generate answer."""
    # MMR retrieval: diversity + relevance
    retriever = vectorstore.as_retriever(
        search_type="mmr",
        search_kwargs={"k": k, "fetch_k": 20}
    )

    docs = retriever.invoke(question)
    context = "\n\n---\n\n".join(d.page_content for d in docs)
    sources = list(set(d.metadata.get("source", "") for d in docs))

    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
    response = llm.invoke([
        SystemMessage(content=f"""Answer the question based on the provided context.
If the context doesn't contain enough information, say so.

Context:
{context}"""),
        HumanMessage(content=question),
    ])

    return f"{response.content}\n\nSources: {', '.join(sources)}"
```

### Multi-Provider LLM Factory

Swap LLM providers without changing application code:

```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic
from langchain_google_vertexai import ChatVertexAI

def get_llm(provider: str = "openai", model: str = None):
    """Factory: returns LangChain LLM for given provider."""
    if provider == "openai":
        return ChatOpenAI(model=model or "gpt-4o-mini", temperature=0)
    elif provider == "anthropic":
        return ChatAnthropic(model=model or "claude-3-5-haiku-20241022")
    elif provider == "vertex":
        return ChatVertexAI(model=model or "gemini-2.5-flash")
    else:
        raise ValueError(f"Unknown provider: {provider}")

# Usage:
llm = get_llm("vertex")  # Switch without changing calling code
```

### DIY RAG Gotchas

**1. crawl4ai vs requests**
`crawl4ai` uses a headless Chromium browser. JavaScript-heavy sites that fail with `requests` work with `crawl4ai`. Install: `pip install crawl4ai && playwright install chromium`.

**2. fit_markdown vs raw_markdown**
`result.markdown_v2.fit_markdown` strips navigation, headers, footers — cleaner for RAG. `result.markdown` is the full page. Use `fit_markdown` for document stores.

**3. Chroma persist_directory**
Without `persist_directory`, Chroma is in-memory and lost on restart. Always set it for production.

**4. Chunk size affects retrieval quality**
1000 tokens with 200 overlap is a starting point. For highly structured API docs, smaller chunks (500 tokens) may retrieve more precisely.

---

## Trade-off Guide

| | Managed RAG (File Search API) | DIY RAG (crawl4ai + LangChain + Chroma) |
|--|-------------------------------|------------------------------------------|
| **Setup time** | 30 min | 2-4 hours |
| **Maintenance** | Low — Google manages everything | High — vector DB, embedding, retrieval tuning |
| **Retrieval control** | None — Google's algorithm | Full — custom chunking, MMR, filters |
| **Cost** | Per query (retrieval + generation) | Embedding cost + hosting |
| **Scale** | Handles thousands of docs easily | Needs tuning at scale |
| **Best for** | Most RAG use cases | When you need custom retrieval or complex filtering |

**Use Managed (File Search API) when:**
- You want something working today
- Standard document Q&A
- You don't need to filter by metadata at query time

**Use DIY (crawl4ai + Chroma) when:**
- You need to scrape and ingest web content
- Custom chunking strategy required
- Complex filtering (by date, author, category) at query time
- You need to see exactly what chunks are being retrieved

---

## What's Next

- Deploy RAG chatbot to Cloud Run → `cloud-run-deployment.md`
- Add Streamlit UI for the chatbot → `streamlit-patterns.md`
- Use File Search API in an ADK agent → `adk-advanced-patterns.md`
