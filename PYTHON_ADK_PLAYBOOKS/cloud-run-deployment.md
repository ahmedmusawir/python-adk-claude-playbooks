# Cloud Run Deployment

## What This Is For

Everything needed to go from local ADK agent to Cloud Run in one session. Complete `deploy.sh`, Dockerfile templates, Secret Manager setup, and the specific flags that prevent AI/streaming workloads from breaking.

---

## Quick Reference — deploy.sh

Copy this, fill in the variables at the top, run it.

```bash
#!/bin/bash
# deploy.sh

# ─── Config ────────────────────────────────────────────────────────────────
PROJECT_ID="your-gcp-project"
REGION="us-east1"
SERVICE_NAME="my-adk-agent"
IMAGE_NAME="gcr.io/${PROJECT_ID}/${SERVICE_NAME}"
# ───────────────────────────────────────────────────────────────────────────

set -e

echo "🚀 Deploying ${SERVICE_NAME}..."

# Build and push image
gcloud builds submit --tag ${IMAGE_NAME} --project ${PROJECT_ID}

# Deploy to Cloud Run
gcloud run deploy ${SERVICE_NAME} \
  --image ${IMAGE_NAME} \
  --region ${REGION} \
  --project ${PROJECT_ID} \
  --platform managed \
  --allow-unauthenticated \
  --port 8080 \
  --memory 2Gi \
  --cpu 2 \
  --min-instances 1 \
  --max-instances 10 \
  --no-cpu-throttling \
  --cpu-boost \
  --timeout 300 \
  --set-env-vars="GOOGLE_CLOUD_PROJECT=${PROJECT_ID},GOOGLE_CLOUD_LOCATION=${REGION},GOOGLE_GENAI_USE_VERTEXAI=1" \
  --set-secrets="OPENAI_API_KEY=openai-api-key:latest" \
  --service-account "my-service@${PROJECT_ID}.iam.gserviceaccount.com"

echo "✅ Deployed. URL:"
gcloud run services describe ${SERVICE_NAME} --region ${REGION} --format="value(status.url)"
```

---

## Source-Based Deploy (Alternative — No Dockerfile Needed)

Cloud Run can build directly from source using Google Cloud Buildpacks. Simpler for Python apps without complex dependencies.

```bash
gcloud run deploy my-agent \
  --source . \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars="GOOGLE_GENAI_USE_VERTEXAI=1,GOOGLE_CLOUD_PROJECT=your-project"
```

Buildpacks detect Python, install `requirements.txt`, and run the app. No Dockerfile needed.

**Use source-based when:** Pure Python app, no Node.js, no complex system deps.
**Use Dockerfile when:** Multi-runtime (Python + Node), custom system packages, Supervisor/Nginx.

---

## Dockerfile — Python ADK Agent

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# System deps (if needed for audio/video processing)
# RUN apt-get update && apt-get install -y ffmpeg && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Shell form CMD — required for $PORT substitution
CMD adk api_server . --port $PORT
```

**Critical:** Use **shell form** `CMD adk api_server . --port $PORT`, NOT JSON array form `CMD ["adk", "api_server", ".", "--port", "$PORT"]`. The JSON array form passes `$PORT` as a literal string, not the environment variable value.

---

## Dockerfile — Streamlit App

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD streamlit run app.py --server.port $PORT --server.address 0.0.0.0
```

Or use a Procfile instead of Dockerfile (simpler for Streamlit):
```
web: streamlit run app.py --server.port $PORT --server.address 0.0.0.0
```

---

## .gcloudignore — What to Exclude

```
# .gcloudignore
.git
.gitignore
.env
*.env
.venv
venv/
__pycache__
*.pyc
*.pyo
node_modules/
*.log
.DS_Store

# Local dev files
.streamlit/secrets.toml
inspect_code/
tests/
```

**Do NOT ignore:**
- `requirements.txt` — build needs it
- `.streamlit/` directory (only ignore `secrets.toml` inside it)
- Any source files your app needs at runtime

---

## Secret Manager Setup

### Store a Secret

```bash
# Create secret
echo -n "your-secret-value" | gcloud secrets create my-secret \
  --data-file=- \
  --project your-project

# Or from a file
gcloud secrets create supabase-url \
  --data-file=./supabase_url.txt

# Add a new version to existing secret
echo -n "new-value" | gcloud secrets versions add my-secret --data-file=-
```

### Use Secrets in Cloud Run

```bash
# Single secret
gcloud run deploy my-service \
  --set-secrets="ENV_VAR_NAME=secret-name:latest"

# Multiple secrets
gcloud run deploy my-service \
  --set-secrets="SUPABASE_URL=supabase-url:latest,SUPABASE_KEY=supabase-key:latest,GHL_API_KEY=ghl-api-key:latest"
```

The `ENV_VAR_NAME=secret-name:latest` format: left side is the env var name your app reads, right side is the Secret Manager secret name + version.

### Convenience Script — Store All Secrets

```bash
#!/bin/bash
# store_secrets.sh

PROJECT="your-project"

echo "Storing secrets in Secret Manager..."

store_secret() {
    local SECRET_NAME=$1
    local SECRET_VALUE=$2

    if gcloud secrets describe "$SECRET_NAME" --project="$PROJECT" &>/dev/null; then
        echo -n "$SECRET_VALUE" | gcloud secrets versions add "$SECRET_NAME" \
            --data-file=- --project="$PROJECT"
        echo "  Updated: $SECRET_NAME"
    else
        echo -n "$SECRET_VALUE" | gcloud secrets create "$SECRET_NAME" \
            --data-file=- --project="$PROJECT" --replication-policy="automatic"
        echo "  Created: $SECRET_NAME"
    fi
}

# Add your secrets here:
store_secret "supabase-url" "https://xxx.supabase.co"
store_secret "supabase-anon-key" "eyJ..."
store_secret "openai-api-key" "sk-..."

echo "Done."
```

---

## Service Account Setup

Create a dedicated service account for your Cloud Run service. Grant only what it needs.

```bash
#!/bin/bash
# grant_permissions.sh

PROJECT="your-project"
SA_NAME="my-agent-sa"
SA_EMAIL="${SA_NAME}@${PROJECT}.iam.gserviceaccount.com"

# Create service account
gcloud iam service-accounts create ${SA_NAME} \
  --display-name="My Agent Service Account" \
  --project=${PROJECT}

# Vertex AI (required for ADK + Gemini/Imagen/TTS)
gcloud projects add-iam-policy-binding ${PROJECT} \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/aiplatform.user"

# Secret Manager (required for --set-secrets)
gcloud projects add-iam-policy-binding ${PROJECT} \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"

# Cloud Storage (if reading GCS prompts or files)
gcloud projects add-iam-policy-binding ${PROJECT} \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/storage.objectViewer"

echo "Service account ready: ${SA_EMAIL}"
```

In deploy.sh, attach it:
```bash
gcloud run deploy my-service \
  --service-account "my-agent-sa@your-project.iam.gserviceaccount.com"
```

---

## Key Flags Explained

### --no-cpu-throttling + --cpu-boost

```bash
--no-cpu-throttling \   # CPU stays allocated between requests (prevents cold processing lag)
--cpu-boost \           # Extra CPU during container startup (faster cold starts)
```

Without `--no-cpu-throttling`, Cloud Run throttles CPU to near zero between requests. For AI/streaming workloads that process asynchronously or maintain connections, this causes timeouts and slow responses. Always set these for ADK agents.

### --min-instances=1

```bash
--min-instances 1 \
```

Keeps at least one instance warm at all times. Without it, the first request after inactivity hits a cold start (10-30 seconds for Python + ADK to load). For production ADK agents, cold starts cause race conditions in session management.

Cost: ~$0.01-0.05/hour for a single warm instance.

### --timeout

```bash
--timeout 300 \     # Max request duration in seconds (default: 300)
```

ADK streaming responses can take 30-120 seconds for complex agent tasks with multiple tool calls. Set to at least 300. Max is 3600.

### --allow-unauthenticated vs Service Account Invoker

```bash
# Public endpoint — no auth required
--allow-unauthenticated

# Private endpoint — caller must have run.invoker role
# (remove --allow-unauthenticated and add invoker to your service accounts)
```

For services only called by other Cloud Run services (e.g., ADK bundle called by ADK wrapper), remove `--allow-unauthenticated` and grant `roles/run.invoker` to the wrapper's service account. Prevents public access.

---

## Health Check Endpoint

Cloud Run's startup probe sends HTTP requests to your service on the configured port. If the service doesn't return a 200 within 240 seconds, Cloud Run marks it unhealthy and kills it.

ADK's `api_server` responds on `/` automatically (returns the API interface), so the startup probe passes without any extra configuration.

For custom FastAPI apps (like the ADK wrapper pattern), add a dedicated health endpoint:

```python
@app.get("/health")
async def health():
    return {"status": "ok"}
```

Then configure the startup probe path in your deploy command if needed:
```bash
--startup-probe-path="/health"
```

---

## Multi-Instance Session Management

`InMemorySessionService` doesn't work across multiple Cloud Run instances. If Cloud Run scales to 2+ instances, different requests may hit different instances, each with its own in-memory session history — conversations break.

**Solution:** Use a database-backed session service:

```bash
# In deploy.sh — add Supabase/Postgres URI as a secret, then:
gcloud run deploy my-agent \
  --set-secrets="DB_URI=session-db-uri:latest"
```

In your startup command:
```bash
CMD adk api_server . --port $PORT --session_service_uri=$DB_URI
```

With a DB-backed session service, any instance can serve any request. Standard for production multi-agent deployments.

For development or single-instance `--min-instances=1` deployments, `InMemorySessionService` is fine.

---

## Complete Example — Full Deploy for ADK Agent

Assume you have:
```
my_project/
├── my_agent/
│   ├── __init__.py
│   └── agent.py
├── Dockerfile
├── requirements.txt
├── .gcloudignore
└── deploy.sh
```

**requirements.txt:**
```
google-adk>=0.1.0
google-cloud-secret-manager
google-cloud-storage
```

**Dockerfile:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD adk api_server . --port $PORT
```

**deploy.sh:**
```bash
#!/bin/bash
PROJECT="my-gcp-project"
REGION="us-east1"
SERVICE="my-adk-agent"

gcloud builds submit --tag gcr.io/${PROJECT}/${SERVICE} --project ${PROJECT}

gcloud run deploy ${SERVICE} \
  --image gcr.io/${PROJECT}/${SERVICE} \
  --region ${REGION} \
  --project ${PROJECT} \
  --allow-unauthenticated \
  --port 8080 \
  --memory 2Gi \
  --cpu 2 \
  --min-instances 1 \
  --no-cpu-throttling \
  --cpu-boost \
  --timeout 300 \
  --set-env-vars="GOOGLE_GENAI_USE_VERTEXAI=1,GOOGLE_CLOUD_PROJECT=${PROJECT},GOOGLE_CLOUD_LOCATION=${REGION}" \
  --service-account "my-agent-sa@${PROJECT}.iam.gserviceaccount.com"

URL=$(gcloud run services describe ${SERVICE} --region ${REGION} --format="value(status.url)")
echo "Deployed: ${URL}"
```

Run: `chmod +x deploy.sh && ./deploy.sh`

---

## Gotchas

**1. CMD shell form for $PORT**
JSON array CMD doesn't expand `$PORT`. Dockerfile CMD must use shell form. Multiple repos burned time on this before the pattern was established.

**2. --no-cpu-throttling for streaming**
Without it, ADK streaming responses time out mid-stream because the CPU gets throttled while waiting for model output. Always set for AI workloads.

**3. --min-instances=1 for session consistency**
With `min-instances=0`, cold starts can take 15-30 seconds. More importantly, session creation race conditions happen when the first request hits an instance that's still warming up. Keep one instance warm.

**4. .gcloudignore is separate from .gitignore**
Cloud Run reads `.gcloudignore`, not `.gitignore`. Both files need to exist. Large repos without `.gcloudignore` upload node_modules and .venv to Cloud Build, causing multi-GB uploads and build timeouts.

**5. Secret Manager version must be specified**
`--set-secrets=VAR=secret-name` without a version fails. Always append `:latest`: `--set-secrets=VAR=secret-name:latest`.

**6. Service account must have Secret Manager access**
Even if you set `--set-secrets`, if the service account doesn't have `roles/secretmanager.secretAccessor`, the deployment succeeds but the service crashes on startup when trying to load secrets.

---

## What's Next

- Expose your agent as a REST API → `fastapi-adk-gateway.md`
- Multi-runtime Python + Node container → `multi-runtime-docker.md`
- Streamlit deployment is the same process → `streamlit-patterns.md`
