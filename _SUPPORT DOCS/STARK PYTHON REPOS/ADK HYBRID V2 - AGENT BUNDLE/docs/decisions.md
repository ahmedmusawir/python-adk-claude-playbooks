# ADK N8N Hybrid Key Decisions

**Project:** google-adk-n8n-hybrid-v2
**Extracted:** 2026-02-23

---

## Decision 1: N8N as Gateway (Not FastAPI or Direct Client)

**Context:**
Need to receive requests from external clients (Streamlit, chatbots, etc.) and route them to ADK agents.

**Decision:**
Use N8N as the webhook gateway layer, with N8N's HTTP Request node forwarding to the ADK Wrapper.

**Alternatives Considered:**
1. **Direct ADK API calls from client** — too complex (requires session creation first)
2. **Custom FastAPI endpoint exposed to public** — more control but more code
3. **N8N** — no custom code for routing; visual workflow editor

**Rationale:**
- N8N handles retries, logging, and workflow visualization for free
- Easy to add preprocessing steps (authentication, transformations) without code
- N8N's Code nodes allow custom JS for edge cases
- Visual workflow = non-engineers can understand and modify the flow

**Trade-offs:**
- ✅ Visual, modifiable routing without code deploys
- ✅ Built-in retry, error handling, logging
- ✅ N8N community nodes for 400+ integrations
- ❌ Another service to maintain (N8N instance)
- ❌ 90s timeout cap in HTTP Request node
- ❌ Debugging is in N8N's interface (not your code)

**Outcome:** Clean separation — N8N handles routing, ADK handles AI. Each can evolve independently.

---

## Decision 2: FastAPI ADK Wrapper (Not Calling ADK Directly)

**Context:**
ADK's native API requires: (1) create session, (2) POST to `/run`, (3) parse streaming response. Every caller would need to implement this.

**Decision:**
Introduce a FastAPI "ADK Wrapper" service that reduces all ADK interaction to one endpoint: `POST /run_agent`.

**Alternatives Considered:**
1. **Clients call ADK directly** — every client must implement session logic
2. **ADK Wrapper** — single point of integration, one implementation of session management

**Rationale:**
- N8N, Streamlit, mobile apps all get the same simple interface
- Session management logic written and tested once
- Can add auth, rate limiting, logging at the wrapper layer
- ADK API format can change; wrapper absorbs the change

**Trade-offs:**
- ✅ All callers use same simple API
- ✅ Session complexity hidden from upstream
- ✅ Single place to add cross-cutting concerns (auth, rate limits)
- ❌ Extra service to deploy and maintain
- ❌ Extra network hop (latency)

**Outcome:** Pattern proven. Every project with multiple clients should have a wrapper layer.

---

## Decision 3: Supabase for Session Persistence (Not In-Memory)

**Context:**
ADK's default session storage is in-memory, which is lost when a Cloud Run container is killed (happens frequently — Cloud Run scales to zero).

**Decision:**
Use `--session_service_uri` to point ADK at a Supabase Postgres database for all session storage.

**Alternatives Considered:**
1. **In-memory (ADK default)** — lost on container restart
2. **Cloud SQL (Google managed Postgres)** — more complex setup, needs VPC or proxy
3. **Supabase** — managed Postgres, easy setup, free tier generous
4. **Redis** — faster but more expensive for session data

**Rationale:**
- Cloud Run containers are ephemeral — scale-to-zero is normal
- Users expect conversation continuity across sessions
- Supabase Postgres is easiest to set up (one connection string)
- ADK manages its own schema — no migrations needed from developers
- Supabase free tier sufficient for most use cases

**Trade-offs:**
- ✅ Sessions survive container restarts
- ✅ Sessions accessible from any container instance (horizontal scaling)
- ✅ Easy setup (just pass the URI)
- ❌ External dependency (if Supabase goes down, sessions fail)
- ❌ Latency per session read/write
- ❌ Cost scales with session volume

**Outcome:** Essential for production. Without this, users lose conversation history on every Cloud Run scale-down.

---

## Decision 4: Source-Based Cloud Run Deployment (Not Local Docker Build)

**Context:**
Need to deploy the ADK bundle to Cloud Run. Two options: build Docker locally and push, or use `gcloud run deploy --source .`.

**Decision:**
Use `--source .` for source-based deployment. Cloud Build handles the Docker build in the cloud.

**Alternatives Considered:**
1. **Local Docker build + push** — `docker build && docker push && gcloud run deploy`
2. **Source-based deploy** — `gcloud run deploy --source .`
3. **CI/CD pipeline (GitHub Actions)** — automated on push

**Rationale:**
- `--source .` is one command vs. three
- No Docker installed locally required
- Cloud Build uses the same Dockerfile (same result)
- `.gcloudignore` controls what gets uploaded

**Trade-offs:**
- ✅ One command to deploy
- ✅ No local Docker required
- ✅ Consistent build environment (Cloud Build)
- ❌ Upload time (source files go to Cloud Storage)
- ❌ Less control over build environment
- ❌ Cloud Build costs (small but non-zero)

**Outcome:** Best developer experience. `./deploy.sh` and done.

---

## Decision 5: Google Secret Manager (Not .env in Container)

**Context:**
Need to store sensitive credentials (Supabase URI, GHL API token) accessible to Cloud Run.

**Decision:**
Store all secrets in Google Secret Manager, inject via `--set-secrets` in deploy command.

**Alternatives Considered:**
1. **Bake secrets into Docker image** — NEVER (image is shareable)
2. **Environment variables in deploy command** — `--set-env-vars="KEY=value"` — appears in deploy history
3. **Google Secret Manager** — dedicated secrets service with versioning, audit logs
4. **HashiCorp Vault** — overkill for this scale

**Rationale:**
- Secrets never appear in command history or code
- Secret Manager has audit logging (who accessed what, when)
- Secret versioning allows rollback
- Cloud Run natively injects them as env vars (no code changes needed)
- IAM control over which service accounts can access which secrets

**Trade-offs:**
- ✅ Secrets never in code or command history
- ✅ Audit trail
- ✅ Versioning + rollback
- ✅ IAM-controlled access
- ❌ One-time setup (create secrets, grant access)
- ❌ Small cost per secret access

**Outcome:** Correct approach for all production deployments. The `store_secrets.sh` script makes the one-time setup painless.

---

## Decision 6: All Vertex AI (Dropped OpenRouter)

**Context:**
Previous ADK experiment (`google-adk-exp-v2`) included LiteLlm + OpenRouter for non-Gemini models (GPT-4, Claude). This production repo dropped it.

**Decision:**
All 5 agents use `model="gemini-2.5-flash"` directly. No LiteLlm, no OpenRouter.

**README note (direct quote):**
> "Because, all other models w/ OpenRouter simply sux!"

**Technical rationale (from observation):**
- OpenRouter adds latency (extra API hop)
- OpenRouter reliability varies
- Vertex AI has guaranteed SLAs on Cloud Run (same network)
- Single auth model — service account handles everything
- Consistent billing (one invoice)

**Trade-offs:**
- ✅ Single auth (service account, no API keys for models)
- ✅ Vertex AI SLAs + reliability
- ✅ Simpler configuration
- ❌ No model flexibility (locked to Gemini)
- ❌ Can't A/B test different LLMs easily

**Outcome:** Pragmatic production decision. For reliability > flexibility, go all-Vertex.

---

## Decision 7: GCS for Hot-Reload Instructions (Not Hardcoded Strings)

**Context:**
Agent instructions (system prompts) determine behavior. Changing them requires code change + redeploy if hardcoded.

**Decision:**
Store all agent instructions as `.txt` files in GCS. Load via callable instruction on every agent run.

**Alternatives Considered:**
1. **Hardcoded string in agent.py** — requires redeploy for every prompt change
2. **GCS loaded at startup** — loaded once, no redeploy but stale until restart
3. **GCS callable (this repo's choice)** — fresh on every request, zero redeploy
4. **Database-stored prompts** — queried on each run; adds DB dependency

**Rationale:**
- Prompt engineering is iterative — change → test → change is the workflow
- GCS callable means any prompt edit is live on the next request
- No code change, no Docker build, no Cloud Run redeploy
- GCS access is fast (same region as Cloud Run)
- GCS has versioning (free rollback of prompts)

**Trade-offs:**
- ✅ Prompt changes are live immediately
- ✅ No redeploy cycle for prompt work
- ✅ GCS versioning = rollback
- ✅ ADC handles auth in Cloud Run (no key management)
- ❌ Every agent run fetches from GCS (latency + cost)
- ❌ If GCS is down, instructions fail (fallback string used)

**Outcome:** Essential for production agents. Prompt iteration speed is critical. Never hardcode instructions.

---

## Decision 8: One Bundle Per Service (Not One Service Per Agent)

**Context:**
5 agents. Architecture choice: one Cloud Run service with all agents, or one service per agent.

**Decision:**
One Cloud Run service running all 5 agents together via `adk api_server .`

**Rationale:**
- `adk api_server .` scans the directory and exposes all agent folders automatically
- One deploy, one service account, one billing unit
- Agents share the same GCS utils (code reuse)
- Inter-agent calls possible (AgentTool) within same bundle

**Trade-offs:**
- ✅ Simple deployment (one service)
- ✅ Shared code (gcs_utils, context_utils)
- ✅ Lower cost (one container)
- ❌ If one agent crashes badly, could affect others
- ❌ Can't scale agents independently
- ❌ One bundle = all agents get same memory/CPU allocation

**Outcome:** Right choice for this scale. When an agent needs independent scaling (e.g., high-traffic ghl_mcp_agent), split it out.

---

## Decision 9: tool_filter on MCPToolset (Optional Restriction)

**Context:**
GoHighLevel MCP server exposes 50+ tools. Exposing all to the LLM causes confusion and increases token usage.

**Decision:**
`tool_filter` parameter on MCPToolset is commented out (all tools available) but documented as an option.

**Pattern:**
```python
MCPToolset(
    connection_params=...,
    # tool_filter=['contacts_get-contact', 'contacts_create-contact', ...]
)
```

**When to activate:**
- Agent starts using wrong tools
- Token costs are high (many tools = large system prompt)
- Want to limit what the agent can do (security/permissions)

**Outcome:** Keep it commented by default; activate when agent misbehaves or costs spike.

---

## Decision 10: ghl_mcp_agent Uses Inline Instruction (Not GCS Callable)

**Context:**
`ghl_mcp_agent` uses a hardcoded `get_rico_instructions(ctx)` function with the instructions as a Python string, while all other agents fetch from GCS.

**Observation:**
This appears to be an intentional exception for this specific agent, likely because:
- The GHL instructions contain dynamic content (references to the specific CRM setup)
- Or this agent was in active development and not yet migrated to GCS

**Pattern difference:**
```python
# Other agents (GCS):
def get_live_instructions(ctx) -> str:
    return fetch_instructions("agent_name")

# ghl_mcp_agent (inline):
def get_rico_instructions(ctx) -> str:
    return """
    Your name is Rico! You are...
    """
```

**Implication for manuals:** Document both patterns. GCS callable is the target state; inline callable is valid for early development or when instructions are dynamic.

---

## Key Decision Themes

**Cross-repo patterns (confirmed again):**
- Single-vendor (all Vertex, all Google Cloud)
- Config-driven (secrets via Secret Manager, not hardcoded)
- Simplicity over flexibility (one bundle, not microservices)

**New themes introduced by this repo:**
1. **Gateway abstraction** — wrap complex APIs with simple endpoints
2. **Ephemeral infrastructure needs persistence layer** — Cloud Run + Supabase
3. **Prompt engineering ≠ code change** — GCS callable instruction
4. **Script-driven ops** — 3 scripts for production lifecycle (permissions, secrets, deploy)

---

_Extracted: 2026-02-23_
