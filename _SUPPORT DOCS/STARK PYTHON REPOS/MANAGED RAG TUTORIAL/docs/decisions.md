# Decisions: managed-rag-google-file-search-api-v1

## Decision 1: Google Managed RAG vs DIY RAG (LangChain + Chroma)

**Choice:** Google's File Search API (managed)

**Context:** This project was explicitly built to learn managed RAG after using crawl4ai + LangChain + Chroma for DIY RAG.

**Rationale:**
- Zero infrastructure: no Chroma, no embedding model, no vector store to manage
- Google handles chunking, embedding creation, indexing, and retrieval
- One API call to upload, one config line to query
- Trade-off: less control (can't tune chunk size, embedding model, retrieval params)

**When to use managed vs DIY:**
| Concern | Managed (File Search) | DIY (LangChain + Chroma) |
|---------|----------------------|--------------------------|
| Setup complexity | Minimal | High |
| Control over chunking | None | Full |
| Control over embedding model | None | Full |
| Vendor lock-in | Google only | Provider-agnostic |
| Local/offline use | No | Yes |
| Cost visibility | Opaque | Transparent |
| Query latency | API call | Local (fast) |

**Conclusion:** Use managed for prototypes and when simplicity matters. Use DIY when you need control, local deployment, or vendor independence.

---

## Decision 2: API Key Auth vs Vertex AI Auth

**Choice:** API key for main lessons, Vertex AI shown in test/old scripts

**Context:** Both use the same `google-genai` SDK, same `client.*` interface.

**Rationale:**
- API key is simpler for learning — no GCP project setup required
- Vertex AI is the production pattern (confirmed in n8n-hybrid and other repos)
- Having both demonstrated in the same repo is educational

**Pattern confirmed:** Production repos converge on Vertex AI (`vertexai=True`). API key is for prototyping/learning.

---

## Decision 3: Lesson-Based File Structure

**Choice:** Separate `lesson_N_*.py` files rather than one script with functions

**Context:** This is a learning/tutorial repo, not a production app.

**Rationale:**
- Each lesson is independently runnable
- Progressive complexity (each lesson builds on previous)
- Reader can see exactly what each stage requires
- Matches the "explain by doing" learning style
- State passed via `store_name.txt` so lessons are decoupled

**Trade-off:** Not reusable as a library. Not suitable for production. This is intentional.

---

## Decision 4: store_name.txt for State Persistence

**Choice:** Plain text file at project root containing store resource path

**Context:** The store ID (`fileSearchStores/name-hash`) needs to be available across multiple scripts.

**Rationale:**
- Simplest possible state — one value, one file
- No database overhead for a single config value
- Same cross-session persistence used by VidGen and crawl4ai (confirmed pattern)

**Note:** In production, this would be an environment variable or config file.

---

## Decision 5: config={'force': True} for Document Deletion

**Choice:** Always pass `config={'force': True}` when deleting documents

**Context:** Discovered through a bug — deletion fails when documents have been processed (chunked).

**Rationale:**
- Without `force=True`, the API raises an error if the document has associated chunks
- Any document that has been uploaded and processed will have chunks
- So `force=True` is effectively always required
- The discovery of this was a key learning moment (visible in code comment: `# ✅ FIXED`)

---

## Decision 6: Pre-generate Structured Summary Before Upload

**Choice:** Use Gemini to analyze the full document first and generate a summary text file, then upload BOTH to the store.

**Context:** Standard RAG chunking breaks large documents into pieces — counting/enumeration questions ("how many X?") fail because the answer spans multiple chunks.

**Rationale:**
- The LLM can read a full PDF in one call (via `types.Part.from_bytes`)
- A structured summary explicitly lists ALL entities (companies, dates, people)
- Uploading summary alongside original gives RAG two representations of the same content
- Summary is optimized for retrieval of aggregate/counting questions

**Pattern:** Pre-process → structured summary file → upload both = better RAG quality for complex questions.

---

## Decision 7: gemini-2.5-flash vs gemini-2.5-pro

**Choice:** Flash for batch queries, Pro for interactive mode

**Context:** Both models work with the File Search tool.

**Rationale:**
- Flash: faster, cheaper, sufficient for most document Q&A
- Pro: better reasoning, used in `query_interactive.py` for live conversation
- Model is swapped by changing one string — easy to test both

**Note:** `query_interactive.py` has `gemini-2.5-pro` uncommented and `gemini-2.5-flash` commented out, showing conscious choice for interactive mode.

---

## Decision 8: requirements.txt (flat) vs pyproject.toml / Poetry

**Choice:** `requirements.txt` (flat pinned)

**Context:** crawl4ai used Poetry + pyproject.toml. This repo uses plain pip + requirements.txt.

**Rationale:**
- Simpler for a tutorial/learning project
- Full version pinning (`google-genai==1.55.0`) locks the exact SDK that was working
- No dev dependencies needed (no tests, no linting in this repo)

**Lesson:** requirements.txt is fine for learning projects. Production projects should use Poetry or similar for dependency management.

---

## Decision 9: No Tests

**Choice:** No pytest or any test files

**Context:** This is a tutorial project, not production code.

**Rationale:** Explicitly a learning repo — the "tests" are the interactive lesson scripts themselves.

**Gap:** Testing patterns for Google File Search API interactions are not yet documented. How to mock `client.file_search_stores` for unit tests is unknown.
