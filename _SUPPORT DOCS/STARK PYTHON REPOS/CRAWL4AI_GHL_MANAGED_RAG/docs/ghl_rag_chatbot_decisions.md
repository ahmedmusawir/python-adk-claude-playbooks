# GHL API RAG Chatbot — Key Decisions

## Decision 1: Google Managed RAG over DIY (LangChain + Chroma)

**Context:** The same repo started with a LangChain + Chroma DIY RAG pipeline.

**Decision:** Replaced with Google File Search API for the GHL chatbot.

**Rationale:**
- Zero embedding management — no chunk_size tuning, no vector store ops
- Free storage; only pay at upload time (one-time embedding cost)
- Implicit retrieval — Gemini decides what to search, not a separate retriever call
- No local vector store to maintain or sync

**Trade-off:** No control over retrieval parameters (can't tune MMR vs similarity, chunk size, etc.). Google is a black box. For a production API chatbot, simplicity wins.

**Rule:** If vendor lock-in is acceptable and speed-to-launch matters, managed RAG is the right call. If retrieval quality needs tuning, DIY.

---

## Decision 2: Dual-Representation Upload (md + SUMMARY per page)

**Context:** Could have uploaded just the `.md` files.

**Decision:** Upload both the raw `.md` AND a structured `_SUMMARY.txt` per page.

**Rationale:**
- Summaries provide structured metadata (endpoint name, HTTP method, URL) that are easy for the master index generator to parse without LLM help
- Raw docs provide parameter details and code examples
- Gemini retrieves from both → breadth from summaries + depth from full docs

**Trade-off:** Doubles the file count (707 → 1414) and upload time. Worth it for retrieval quality.

---

## Decision 3: Synthetic Master Index File

**Context:** "List all categories" queries return garbage from standard RAG (answer is distributed across 700+ chunks).

**Decision:** Generate a single aggregated document (`GHL_API_MASTER_INDEX_V2.txt`) and upload it to the same store.

**Rationale:**
- A single document containing all categories + endpoint counts + HTTP methods can be retrieved in one chunk
- Eliminates the multi-document aggregation problem completely
- Acts as the store's "table of contents"

**Trade-off:** Must be manually regenerated if the docs update. Acceptable — this is a static API docs snapshot.

**Alternative rejected:** Prompt engineering alone (telling Gemini to search multiple docs and aggregate). Too slow, unreliable.

---

## Decision 4: Filename-Based Category Extraction (v2) over LLM-Parsed (v1)

**Context:** Master index v1 used LLM to extract category from each summary. Generated 290 categories instead of 38.

**Decision:** v2 parses the filename and maps to a ground truth dictionary.

**Rationale:**
- Filenames follow a deterministic pattern: `...-docs-ghl-{CATEGORY}-{endpoint}`
- Ground truth dict (`TRUE_CATEGORIES`) maps to canonical category names
- Zero LLM calls needed — runs in seconds, not minutes
- 100% consistent categorization

**Lesson:** For structured metadata extraction from URL/filename patterns, regex + lookup table beats LLM every time. LLMs over-split, hallucinate subcategories, and are inconsistent.

---

## Decision 5: System Prompt Encodes Retrieval Strategy

**Context:** Generic "helpful assistant" system prompt gave poor results on list/counting queries.

**Decision:** System prompt explicitly specifies search strategies per query type:
- "List all" → search "MASTER_INDEX" first
- "How many" → search "STATISTICS"
- Specific endpoints → try variations (create/add, get/fetch)

**Rationale:**
- Gemini uses File Search as a tool — it picks search terms. Without guidance, it picks bad terms.
- Making the search strategy explicit in the prompt dramatically improves first-try retrieval success
- Users get consistent, well-formatted responses

**Key insight:** The system prompt is not just about tone — it's the retrieval algorithm.

---

## Decision 6: Split-Panel UI Layout (v2)

**Context:** v1 used standard Streamlit chat bubbles (full response inline in the chat).

**Decision:** v2 uses split panel: chat history (left, 1/3 width) + full response (right, 2/3 width).

**Rationale:**
- API docs responses are long (parameter tables, code blocks)
- Standard chat bubble format truncates or requires scrolling
- Right panel gives a wide reading area for formatted markdown
- Left panel keeps interaction flow clean

**Trade-off:** Custom layout requires more CSS/layout code. Complexity is worth it for readability of long responses.

---

## Decision 7: Stateless Queries (No Conversation History Passed)

**Context:** Both chatbot versions maintain `st.session_state.messages` but don't pass history to Gemini.

**Decision:** Each query is: `f"{SYSTEM_PROMPT}\n\nUser Question: {prompt}"` — stateless.

**Rationale (inferred):** This is an API documentation lookup tool, not a conversational assistant. Users ask independent questions ("how do I create a contact?", "what are the calendar endpoints?"). Conversation context doesn't add value — it would just consume tokens.

**Trade-off:** Multi-turn flows ("now show me the parameters for that") don't work. For pure lookup, this is fine.

---

## Decision 8: API Key Mode (Not Vertex AI) Throughout

**Context:** Other repos (adk-exp-v2, n8n-hybrid) use `GOOGLE_GENAI_USE_VERTEXAI=TRUE` in production.

**Decision:** This repo uses `genai.Client(api_key=...)` only — no Vertex AI.

**Rationale:** This is a personal tooling project, not a deployed service. API key mode is simpler, cheaper for personal use, and doesn't require GCP project setup. No Cloud Run deployment means no need for ADC.

**Rule:** Personal tools / local dev → API key. Deployed services → Vertex AI.

---

## Decision 9: Confirmation Gate Before Batch Upload

**Context:** Batch upload is irreversible (can't bulk-delete easily) and takes 20-30 minutes.

**Decision:** `input("⚡ Proceed with batch upload? (y/n)")` with full stats displayed before confirming.

**Rationale:**
- Shows file count, total size, estimated time, cost warning before proceeding
- Running the script twice accidentally would duplicate all 1414 documents
- Consistent with the "confirmation gate" pattern seen across all repos

**The gate displays:**
```
📊 Files to upload:
   • Markdown files: 707
   • Summary files: 707
   • Total files: 1414
   • Total size: X MB
⏱️  Estimated time: 59 minutes
```

---

## Decision 10: Three Master Index Versions

**Context:** Had to iterate on the master index quality.

**Decision:** Keep all three versions in `outputs/pages/`:
- `GHL_API_MASTER_INDEX_SUMMARY.txt` — raw v1 output (LLM-parsed, 290 categories)
- `GHL_API_MASTER_INDEX_CLEAN.txt` — cleaned v1 (normalized categories)
- `GHL_API_MASTER_INDEX_V2.txt` — v2 (filename-based, 38 categories, best)

**Rationale:**
- Keeping old versions shows the evolution (learning artifact)
- V2 only gets uploaded to the store; v1 files are local reference
- `cleanup_master_index.py` shows the normalization logic if v2 approach ever needs to revert

**Lesson:** Don't delete iterative attempts. They document what failed and why.

---

## Decision 11: Store ID in Plaintext File (not env var)

**Context:** The File Search store resource path (`fileSearchStores/ghlapiv2docs-snonl43ig3vp`) needs to be shared across all 12+ scripts.

**Decision:** `ghl_store_name.txt` — a plaintext file at project root, read by every script.

**Rationale:**
- Not a secret (it's a reference to a cloud resource, not a credential)
- File-based = survives across terminals, sessions, restarts
- All scripts use identical read pattern: `Path("ghl_store_name.txt").read_text().strip()`

**Alternative rejected:** Hard-coding in each script. Would break when store is recreated.

**Pattern confirmed cross-repo:** 4 repos now use this file-for-state approach for non-secret persistent values.
