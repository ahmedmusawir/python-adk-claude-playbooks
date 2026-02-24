# N8N Integration & Cloud Run Deployment Guide

**Project:** google-adk-n8n-hybrid-v2
**Extracted:** 2026-02-23

---

## Overview

This document covers two tightly coupled topics in this repo:
1. **N8N as a gateway** — how N8N connects external clients to ADK agents
2. **Production Cloud Run deployment** — secrets, IAM, source-based deploy

These are the primary new patterns not seen in previous repos. Both are essential for production ADK deployments.

---

## Part 1: N8N Integration

### What N8N Is (In This Context)

N8N is a self-hosted workflow automation tool. In this project, it acts as a **lightweight API gateway** — receiving webhooks from any client, normalizing the payload, calling the ADK Wrapper, and returning the response.

The actual N8N workflow is in `Adk_N8N_Hybrid_v4.json` and can be imported directly into any N8N instance.

---

### The N8N Workflow (4 Nodes)

```
[Webhook] → [Code Node 1] → [HTTP Request] → [Code Node 2] → [Respond to Webhook]
```

#### Node 1: Webhook

```
Type: Webhook
Method: POST
Path: f11820f4-aaf0-4bb8-b536-b9097cc67877  (unique UUID — change for your instance)
Response Mode: responseNode   ← REQUIRED — tells N8N to wait for response before replying
```

**The webhook URL format:**
```
https://your-n8n-instance.com/webhook/f11820f4-aaf0-4bb8-b536-b9097cc67877
```

Clients call this URL with a POST body.

---

#### Node 2: Code Node 1 (Normalize Payload)

```javascript
// Extract fields from webhook body and clean up for the HTTP Request
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

**Why:** N8N wraps the webhook body inside `.body`. This node extracts and flattens the fields for the HTTP Request node.

---

#### Node 3: HTTP Request (Call ADK Wrapper)

```
Method: POST
URL: http://localhost:8080/run_agent    ← ADK Wrapper
Body Type: Body Parameters
Parameters:
  agent_name  =  {{ $json.agent_name }}
  message     =  {{ $json.message }}
  user_id     =  {{ $json.user_id }}
  session_id  =  {{ $json.session_id }}
Options:
  Timeout: 90000 ms   ← 90 seconds — LLM calls can be slow
```

**Note on URL:** `localhost:8080` assumes N8N and ADK Wrapper run on the same machine/container. In production, this would be the internal Cloud Run URL or a VPC internal address.

---

#### Node 4: Code Node 2 (Package Response)

```javascript
// Package the agent wrapper's response for the webhook response
const agentResponse = items[0].json;

const responseObject = {
  message: agentResponse.response,
  session_id: agentResponse.session_id
};

// JSON.stringify handles special characters (quotes, newlines) correctly
const finalJsonString = JSON.stringify(responseObject);
return [{ json: { data: finalJsonString } }];
```

**Why JSON.stringify:** Agent responses can contain quotes, newlines, JSON snippets. Stringifying prevents N8N from misinterpreting the data.

---

#### Node 5: Respond to Webhook

Default settings. This node sends the final data back to the caller that hit the webhook URL.

---

### Client Request Format

What any client needs to send to the N8N webhook:

```json
POST https://your-n8n.com/webhook/{path}
Content-Type: application/json

{
  "agent_name": "jarvis_agent",
  "message": "What is the weather in New York?",
  "userId": "user_123",
  "session_id": "session_abc"
}
```

**Field notes:**
- `userId` — capitalized (N8N webhook quirk — normalize in Code Node 1)
- `session_id` — controls conversation continuity. Same session_id = same conversation
- `agent_name` — must match the folder name of the agent in the ADK bundle

---

### Response Format

What the caller receives back:

```json
{
  "data": "{\"message\":\"It's 72°F and sunny in New York.\",\"session_id\":\"session_abc\"}"
}
```

**Note:** `data` is a JSON string (double-encoded). Parse it before using:
```javascript
const parsed = JSON.parse(response.data);
const agentReply = parsed.message;
const sessionId = parsed.session_id;
```

---

## Part 2: Cloud Run Deployment

### Overview

This is a **source-based deployment** — you push your code to Cloud Build which builds the Docker image and deploys it to Cloud Run. No local Docker build required.

### Prerequisites

```bash
# One-time setup
gcloud auth login
gcloud config set project your-project-id
gcloud services enable run.googleapis.com cloudbuild.googleapis.com secretmanager.googleapis.com
```

---

### Step 1: Grant IAM Permissions (One-Time)

The Cloud Run service needs a service account with permissions to call Vertex AI and read from GCS.

```bash
# grant_permissions.sh
SERVICE_ACCOUNT_EMAIL="your-sa@your-project.iam.gserviceaccount.com"
PROJECT_ID="your-project-id"

# Required for Vertex AI / Gemini
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$SERVICE_ACCOUNT_EMAIL" \
  --role="roles/aiplatform.user"

# Required for GCS (fetch instructions + context)
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$SERVICE_ACCOUNT_EMAIL" \
  --role="roles/storage.objectViewer"

# Also needed if using Secret Manager in deploy.sh:
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$SERVICE_ACCOUNT_EMAIL" \
  --role="roles/secretmanager.secretAccessor"
```

Run once per service account. Run again only when adding new required permissions.

---

### Step 2: Store Secrets (One-Time or When Secrets Change)

```bash
# store_secrets.sh — reads .env and pushes to Secret Manager
./store_secrets.sh
```

The script:
1. Reads `.env` for the values
2. Creates the secret in Secret Manager if it doesn't exist
3. Adds the value as a new version

After this, your `.env` is only needed locally. Cloud Run gets secrets from Secret Manager.

**Secret Manager naming convention (from this repo):**
```
SUPABASE_DB_URI  →  adk-db-uri
GHL_API_TOKEN    →  ghl-api-key
GHL_LOCATION_ID  →  ghl-location-id
```

---

### Step 3: Ensure .gcloudignore Exists

Before deploying, verify `.gcloudignore` contains these entries:

```
.venv/          # CRITICAL — don't upload gigabytes of virtualenv
__pycache__/
*.pyc
.git/
.env            # CRITICAL — local secrets, never upload
*.json          # CRITICAL — service account keys
*.log
```

Without `.gcloudignore`, `--source .` uploads everything in the directory to Cloud Storage before Cloud Build — including secrets and virtualenvs.

---

### Step 4: Deploy

```bash
# deploy.sh
SERVICE_NAME="your-service-name"
REGION="us-east1"
PROJECT_ID="your-project-id"
GCS_BUCKET_NAME="your-gcs-bucket"
SERVICE_ACCOUNT_EMAIL="your-sa@your-project.iam.gserviceaccount.com"

gcloud run deploy "$SERVICE_NAME" \
  --source . \
  --region="$REGION" \
  --project="$PROJECT_ID" \
  --allow-unauthenticated \
  --service-account="$SERVICE_ACCOUNT_EMAIL" \
  --set-secrets="DB_URI=adk-db-uri:latest,GHL_API_TOKEN=ghl-api-key:latest,GHL_LOCATION_ID=ghl-location-id:latest" \
  --set-env-vars="GOOGLE_CLOUD_PROJECT=${PROJECT_ID},GCS_BUCKET_NAME=${GCS_BUCKET_NAME},GOOGLE_CLOUD_LOCATION=${REGION},GOOGLE_GENAI_USE_VERTEXAI=TRUE"
```

**What happens:**
1. Source files (minus `.gcloudignore`) uploaded to Cloud Storage
2. Cloud Build runs `docker build` using your `Dockerfile`
3. Image pushed to Artifact Registry
4. Cloud Run service updated with new image + env vars + secrets

---

### Understanding --set-secrets

```bash
--set-secrets="DB_URI=adk-db-uri:latest,GHL_API_TOKEN=ghl-api-key:latest"
```

Format: `ENV_VAR_NAME=secret-manager-name:version`

The container sees `$DB_URI` as a normal environment variable. The value comes from Secret Manager at container startup. `latest` always uses the most recent version.

**Security benefit:** The secret value never appears in your deploy command, Cloud Build logs, or Cloud Run service configuration page.

---

### The Critical Dockerfile Detail

```dockerfile
# ❌ WRONG — exec form, $PORT is never substituted
CMD ["adk", "api_server", ".", "--port=${PORT}", "--session_service_uri=${DB_URI}"]

# ✅ CORRECT — shell form, bash substitutes $PORT and $DB_URI at runtime
CMD adk api_server . --host=0.0.0.0 --port=${PORT} --session_service_uri=${DB_URI}
```

Cloud Run sets `$PORT` at container startup. The exec form (`CMD [...]`) bypasses the shell, so `${PORT}` is treated as a literal string. The shell form (`CMD string`) runs through `/bin/sh -c` which performs variable substitution.

**Both `$PORT` and `$DB_URI` must be substituted at runtime.** Shell form is required.

---

### Local Development Setup

```bash
# start_server.sh
source .env   # loads SUPABASE_DB_URI

adk api_server . \
  --host=0.0.0.0 \
  --port=8000 \
  --session_service_uri="$SUPABASE_DB_URI"
```

The `.env` file provides the Supabase URI locally. The same URI is stored in Secret Manager for production.

**Local auth:** `gcloud auth application-default login` provides credentials for Vertex AI and GCS. The service account JSON (`cyberize-vertex-api.json`) is an alternative for local dev when ADC isn't set up.

---

## Data Flow Summary

### Session ID Lifecycle

```
New user:
  client sends:    { session_id: null or "new" }
  N8N sends:       { session_id: null }
  Wrapper creates: new UUID session in Supabase
  Wrapper returns: { session_id: "abc-123-..." }
  client stores:   session_id for future requests

Returning user:
  client sends:    { session_id: "abc-123-..." }
  ADK fetches:     conversation history from Supabase
  Agent responds:  with context from previous messages
```

### Instruction Fetch on Every Request

```
Client sends request
    ↓
ADK calls agent
    ↓
ADK calls get_live_instructions(ctx)
    ↓
fetch_instructions("agent_name") → GCS API call
    ↓
Returns latest instruction text from:
  gs://bucket/ADK_Bundle/agent_name/agent_name_instructions.txt
    ↓
Agent runs with those instructions
```

**Cost:** One GCS read per agent call. Cost is minimal (~$0.0004 per 10,000 reads) but is a real network call.

---

## Troubleshooting Common Issues

### Agent gets $PORT as literal string
**Problem:** Dockerfile uses exec form `CMD [...]`
**Fix:** Change to shell form `CMD adk api_server ...`

### Sessions lost on container restart
**Problem:** `--session_service_uri` not set, using in-memory default
**Fix:** Add `--session_service_uri=${DB_URI}` to CMD and ensure DB_URI is set

### Instructions not updating after GCS edit
**Problem:** This shouldn't happen with callable instructions — they fetch fresh on every run
**Check:** Verify the agent uses `instruction=get_live_instructions` not `instruction=get_live_instructions()`

### Cloud Build uploads .env or .venv
**Problem:** Missing or incomplete `.gcloudignore`
**Fix:** Ensure `.gcloudignore` has `.env`, `.venv/`, `*.json`

### Service account auth fails in Cloud Run
**Problem:** Service account doesn't have the required IAM roles
**Fix:** Run `grant_permissions.sh` again with the missing role

---

_Extracted: 2026-02-23_
