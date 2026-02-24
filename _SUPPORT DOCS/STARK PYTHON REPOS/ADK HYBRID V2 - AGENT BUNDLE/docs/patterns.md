# ADK N8N Hybrid Patterns

**Project:** google-adk-n8n-hybrid-v2
**Extracted:** 2026-02-23

---

## Overview

This document captures the copy-pasteable patterns from this production ADK deployment. Key additions vs. previous repos: N8N integration, Secret Manager deployment, Supabase session persistence, and a clean two-function GCS context pattern.

---

## Pattern 1: Minimal Agent Definition (Production Form)

Every agent in this repo follows this exact minimal pattern:

```python
# {agent_name}/agent.py
from google.adk.agents import Agent
from google.adk.tools import FunctionTool          # if using GCS context tool
from utils.gcs_utils import fetch_instructions

def get_live_instructions(ctx) -> str:
    """Called on every agent run — fetches fresh instructions from GCS."""
    return fetch_instructions("agent_name")

root_agent = Agent(
    name="agent_name",           # underscore format
    model="gemini-2.5-flash",    # Vertex AI model (no LiteLlm needed)
    description="One-line description",
    instruction=get_live_instructions,  # callable, not string
    tools=[...]
)
```

```python
# {agent_name}/__init__.py
from .agent import root_agent   # ADK discovery mechanism
```

**Key:** `instruction=get_live_instructions` — pass the function object, not the return value. The `()` would evaluate once at startup; no `()` means ADK calls it fresh on every run.

---

## Pattern 2: GCS Callable Instructions (Hot-Reload)

Full pattern for fetching agent instructions from GCS on every run:

```python
# utils/gcs_utils.py
from google.cloud import storage

BUCKET_NAME = "your-bucket-name"
BASE_FOLDER = "ADK_Agent_Bundle_1"

def fetch_instructions(agent_name: str) -> str:
    """
    Fetches agent instructions from GCS.
    Path: {BASE_FOLDER}/{agent_name}/{agent_name}_instructions.txt
    """
    try:
        storage_client = storage.Client()
        bucket = storage_client.bucket(BUCKET_NAME)
        file_path = f"{BASE_FOLDER}/{agent_name}/{agent_name}_instructions.txt"
        blob = bucket.blob(file_path)
        instructions = blob.download_as_text(encoding='utf-8')
        print(f"Successfully fetched instructions for '{agent_name}' from GCS.")
        return instructions
    except Exception as e:
        print(f"ERROR: Could not fetch instructions for '{agent_name}'. Error: {e}")
        return f"Error: Could not load instructions for {agent_name}."
```

**GCS file path convention:**
```
{bucket}/{bundle_folder}/{agent_name}/{agent_name}_instructions.txt
```

**Why this pattern wins:**
- Change agent behavior → edit the .txt file in GCS → live on next request
- Zero redeployment needed for prompt changes
- Works automatically in Cloud Run (ADC handles auth)

---

## Pattern 3: GCS Knowledge Base FunctionTool

Two-function pattern for on-demand document retrieval from GCS:

```python
# utils/context_utils.py
from google.cloud import storage
from google.cloud.exceptions import NotFound

BUCKET_NAME = "your-bucket-name"
FOLDER_NAME = "ADK_Agent_Bundle_1/context_store"

def fetch_context(file_name: str) -> str:
    """
    Reads the full text of a specified context file from the knowledge base.

    Use this to answer any questions about products by passing
    'PRODUCTS.md' as the file_name.

    Args:
        file_name: The name of the context file to fetch (e.g., "PRODUCTS.md").

    Returns:
        The content of the file as a string, or an error message if not found.
    """
    try:
        storage_client = storage.Client()
        bucket = storage_client.bucket(BUCKET_NAME)
        blob_path = f"{FOLDER_NAME}/{file_name}"
        blob = bucket.blob(blob_path)
        print(f"--- FETCHING CONTEXT: {blob_path} from GCS ---")
        content = blob.download_as_text()
        return content
    except NotFound:
        return f"ERROR: Context file '{file_name}' not found in the knowledge base."
    except Exception as e:
        return f"An unexpected error occurred while fetching context file '{file_name}': {e}"
```

**Usage in agent:**
```python
product_context_tool = FunctionTool(func=fetch_context)

root_agent = Agent(
    ...
    tools=[product_context_tool]
)
```

**Critical detail:** The docstring IS the tool description. ADK reads it to know when to call the tool and what argument to pass. Write docstrings as instructions to the LLM, not as code docs.

---

## Pattern 4: Custom FunctionTool (Safe Eval Calculator)

Pattern for a custom Python function as an ADK tool:

```python
# calc_agent/agent.py
def calculate(expression: str) -> float:
    """
    Calculator tool for mathematical expressions.
    Supports: +, -, *, /, **, sqrt, sin, cos, tan, log

    Example: "2 + 2" or "sqrt(16) * 3"
    """
    import math

    expression = expression.replace('sqrt', 'math.sqrt')
    expression = expression.replace('sin', 'math.sin')
    expression = expression.replace('cos', 'math.cos')
    expression = expression.replace('tan', 'math.tan')
    expression = expression.replace('log', 'math.log')

    allowed_names = {
        'math': math,
        'abs': abs,
        'round': round,
        'min': min,
        'max': max,
    }

    try:
        result = eval(expression, {"__builtins__": {}}, allowed_names)
        return float(result)
    except Exception as e:
        raise ValueError(f"Cannot calculate '{expression}': {str(e)}")

root_agent = Agent(
    ...
    tools=[calculate],  # Pass function directly — ADK wraps it in FunctionTool automatically
)
```

**Security note:** `eval(expr, {"__builtins__": {}}, allowed_names)` — the empty `__builtins__` dict prevents access to `import`, `open`, `exec`, etc. Only the `allowed_names` dict is accessible.

---

## Pattern 5: MCPToolset for External API (GHL CRM)

```python
# ghl_mcp_agent/agent.py
import os
from google.adk.agents import Agent
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, StreamableHTTPConnectionParams

GHL_API_TOKEN = os.getenv("GHL_API_TOKEN", "your_token_here")
GHL_LOCATION_ID = os.getenv("GHL_LOCATION_ID", "your_location_id_here")

root_agent = Agent(
    name="ghl_mcp_agent",
    model="gemini-2.5-flash",
    description="CRM agent with live GHL access",
    instruction=get_instructions,
    tools=[
        MCPToolset(
            connection_params=StreamableHTTPConnectionParams(
                url="https://services.leadconnectorhq.com/mcp/",
                headers={
                    "Authorization": f"Bearer {GHL_API_TOKEN}",
                    "locationId": GHL_LOCATION_ID
                }
            ),
            # Optional: limit exposed tools to reduce LLM confusion
            # tool_filter=['contacts_get-contact', 'contacts_create-contact']
        )
    ],
)
```

**Pattern notes:**
- `os.getenv()` with fallback string — credentials from env, never hardcoded
- `tool_filter` — comment it out to allow all tools; uncomment to restrict
- Headers passed to MCP server (auth + tenant ID pattern for SaaS CRMs)

---

## Pattern 6: N8N Workflow — Webhook → Agent → Response

The complete 4-node N8N workflow for connecting any client to ADK:

**Node 1: Webhook** (receives requests from frontend/client)
```
Type: Webhook
Method: POST
Path: {unique-uuid}
Response Mode: responseNode  ← must be set for async response
```

**Node 2: Code Node 1** (normalize payload)
```javascript
// Normalize incoming webhook body to clean agent request
const agentName = items[0].json.body.agent_name;
const message = items[0].json.body.message;
const userId = items[0].json.body.userId;
const sessionId = items[0].json.body.session_id;

const newOutput = {
  agent_name: agentName,
  message: message,
  user_id: userId,
  session_id: sessionId
};

items[0].json = newOutput;
return items;
```

**Node 3: HTTP Request** (call ADK Wrapper)
```
Method: POST
URL: http://localhost:8080/run_agent   ← ADK Wrapper endpoint
Body Parameters:
  agent_name: ={{ $json.agent_name }}
  message: ={{ $json.message }}
  user_id: ={{ $json.user_id }}
  session_id: ={{ $json.session_id }}
Options:
  timeout: 90000   ← 90 seconds (LLM calls can be slow!)
```

**Node 4: Code Node 2** (package response)
```javascript
// Package agent response for downstream
const agentResponse = items[0].json;

const responseObject = {
  message: agentResponse.response,
  session_id: agentResponse.session_id
};

// JSON.stringify handles special characters safely
const finalJsonString = JSON.stringify(responseObject);
return [{ json: { data: finalJsonString } }];
```

**Node 5: Respond to Webhook** (sends response back to caller)
```
Type: Respond to Webhook
(default settings)
```

**Data contract between N8N and ADK Wrapper:**
```
Inbound (N8N → Wrapper):     { agent_name, message, user_id, session_id }
Outbound (Wrapper → N8N):    { response, session_id }
```

---

## Pattern 7: ADK Wrapper API Contract

What the FastAPI wrapper must implement to be compatible with this N8N flow:

```python
# ADK Wrapper endpoint contract (FastAPI)
# POST /run_agent

# Request body:
{
    "agent_name": "jarvis_agent",   # which agent to call
    "message": "What is 2+2?",      # user's message
    "user_id": "user_123",          # identifies the user
    "session_id": "session_abc"     # identifies the conversation
}

# Response body:
{
    "response": "2+2 equals 4.",    # agent's reply
    "session_id": "session_abc"     # echoed back for continuity
}
```

---

## Pattern 8: Dockerfile for Cloud Run (Shell Form vs Exec Form)

**CRITICAL — Two forms of CMD behave differently:**

```dockerfile
# ❌ EXEC FORM — Does NOT substitute env vars ($PORT stays literal)
CMD ["adk", "api_server", ".", "--port=${PORT}"]

# ✅ SHELL FORM — Shell substitutes $PORT before running the command
CMD adk api_server . --host=0.0.0.0 --port=${PORT} --session_service_uri=${DB_URI}
```

**Full production Dockerfile:**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD adk api_server . --host=0.0.0.0 --port=${PORT} --session_service_uri=${DB_URI}
```

**Why:** Cloud Run injects `$PORT` at runtime. Shell form lets the shell substitute it. Exec form treats `${PORT}` as a literal string.

---

## Pattern 9: .gcloudignore (Secure Source-Based Deploy)

```
# .gcloudignore — prevents sensitive/unnecessary files from going to Cloud Build

.venv/          # huge virtual env — DO NOT upload
__pycache__/
*.pyc
.git/
.gitignore
.env            # LOCAL secrets — NEVER upload
*.log
*.json          # service account keys (*.json) — NEVER upload
```

**Critical:** Without `.gcloudignore`, `gcloud run deploy --source .` would upload your entire `.venv/` (gigabytes), your `.env` secrets, and any service account JSON files to Cloud Build.

---

## Pattern 10: Google Secret Manager Injection via deploy.sh

```bash
gcloud run deploy "$SERVICE_NAME" \
  --source . \
  --region="$REGION" \
  --project="$PROJECT_ID" \
  --allow-unauthenticated \
  --service-account="$SERVICE_ACCOUNT_EMAIL" \
  --set-secrets="DB_URI=adk-db-uri:latest,GHL_API_TOKEN=ghl-api-key:latest,GHL_LOCATION_ID=ghl-location-id:latest" \
  --set-env-vars="GOOGLE_CLOUD_PROJECT=${PROJECT_ID},GCS_BUCKET_NAME=${GCS_BUCKET_NAME},GOOGLE_CLOUD_LOCATION=${REGION},GOOGLE_GENAI_USE_VERTEXAI=TRUE"
```

**Key flags:**
- `--set-secrets` — maps Secret Manager secrets to env var names in the container
  - Format: `ENV_VAR_NAME=secret-manager-name:version`
  - `latest` means always use newest version
- `--set-env-vars` — non-secret config values
- `GOOGLE_GENAI_USE_VERTEXAI=TRUE` — switches ADK from API key mode to Vertex AI mode
- `--service-account` — explicit identity (not the default compute SA)

---

## Pattern 11: store_secrets.sh (Secret Manager Population)

```bash
#!/bin/bash
set -e

PROJECT_ID="your-project-id"
ENV_FILE=".env"

# Format: "ENV_VAR_NAME/secret-manager-name"
SECRETS_MAP=(
  "SUPABASE_DB_URI/adk-db-uri"
  "GHL_API_TOKEN/ghl-api-key"
  "GHL_LOCATION_ID/ghl-location-id"
)

for mapping in "${SECRETS_MAP[@]}"; do
  ENV_VAR_NAME="${mapping%%/*}"
  SECRET_NAME="${mapping##*/}"

  VALUE=$(grep -v '^#' "$ENV_FILE" | grep "$ENV_VAR_NAME" | cut -d '=' -f2-)

  if [ -z "$VALUE" ]; then
    echo "Warning: $ENV_VAR_NAME not found in $ENV_FILE. Skipping."
    continue
  fi

  # Create secret if it doesn't exist
  if ! gcloud secrets describe "$SECRET_NAME" --project="$PROJECT_ID" &>/dev/null; then
    gcloud secrets create "$SECRET_NAME" --replication-policy="automatic" --project="$PROJECT_ID"
  fi

  # Add new version
  echo -n "$VALUE" | gcloud secrets versions add "$SECRET_NAME" --data-file=- --project="$PROJECT_ID"
done
```

**Key technique:** `echo -n "$VALUE" | gcloud secrets versions add ... --data-file=-`
- `echo -n` — no trailing newline (important for connection strings)
- `--data-file=-` — reads secret value from stdin

---

## Pattern 12: Supabase Session URI for ADK

```bash
# Local development (start_server.sh)
adk api_server . \
  --host=0.0.0.0 \
  --port=8000 \
  --session_service_uri="$SUPABASE_DB_URI"

# Cloud Run (Dockerfile CMD)
CMD adk api_server . --host=0.0.0.0 --port=${PORT} --session_service_uri=${DB_URI}
```

**Session URI format:**
```
postgresql://postgres.{project_ref}:{password}@aws-0-{region}.pooler.supabase.com:6543/postgres
```

**What it does:** ADK automatically creates tables in this Postgres database to store conversation history, session state, and agent memory. No schema management needed — ADK handles it.

---

## Pattern 13: Vertex AI Toggle (GOOGLE_GENAI_USE_VERTEXAI)

Two ways ADK can call Gemini:
```bash
# Mode 1: API Key (local dev, simple)
GOOGLE_GENAI_USE_VERTEXAI=FALSE
GOOGLE_API_KEY=your_api_key

# Mode 2: Vertex AI (production, Cloud Run)
GOOGLE_GENAI_USE_VERTEXAI=TRUE
# + service account with roles/aiplatform.user
# No API key needed — ADC handles auth automatically
```

**Agent code is identical in both modes** — just the env vars differ.

In Cloud Run: `GOOGLE_GENAI_USE_VERTEXAI=TRUE` is set in `--set-env-vars`. The service account handles all auth automatically.

---

## Patterns Summary Table

| Pattern | File | Key Learning |
|---------|------|-------------|
| Minimal agent | `*/agent.py` | 5 fields + callable instruction |
| GCS hot-reload instructions | `utils/gcs_utils.py` | callable = fresh on every run |
| GCS knowledge base tool | `utils/context_utils.py` | docstring = tool description for LLM |
| Custom function tool | `calc_agent/agent.py` | safe eval with restricted builtins |
| MCPToolset | `ghl_mcp_agent/agent.py` | StreamableHTTP + headers for auth |
| N8N 4-node workflow | `Adk_N8N_Hybrid_v4.json` | Webhook→Code→HTTP→Code→Respond |
| Dockerfile shell form | `Dockerfile` | Shell form required for $PORT substitution |
| .gcloudignore | `.gcloudignore` | Prevents .env/.venv from upload |
| Secret Manager injection | `deploy.sh` | --set-secrets maps to env vars |
| store_secrets.sh | `store_secrets.sh` | .env → Secret Manager via stdin |
| Supabase sessions | `start_server.sh` + `Dockerfile` | --session_service_uri for persistence |
| Vertex toggle | `.env_example` + `deploy.sh` | GOOGLE_GENAI_USE_VERTEXAI env var |

---

_Extracted: 2026-02-23_
