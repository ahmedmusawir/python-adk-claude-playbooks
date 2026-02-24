Of course. Based on our entire deployment process, here is the comprehensive documentation for deploying the ADK Agent Bundle.

---

## 2\. `deployment.md`

Create a new file named `deployment.md` inside your `/docs` directory and add the following content.

````markdown
# ADK Agent Bundle Deployment Guide

This document provides a step-by-step guide for deploying the multi-agent ADK bundle to **Google Cloud Run**. The process uses a source-based deployment with **Cloud Build**, secure authentication via an **IAM Service Account**, and secret management with **Google Secret Manager**.

---

## Prerequisites

Before you begin, ensure you have the following:

1.  **Google Cloud SDK (`gcloud` CLI):** Installed and authenticated. Run `gcloud auth login` and `gcloud config set project [YOUR_PROJECT_ID]`.
2.  **Project Source Code:** The complete project directory with all agent code and configuration files.
3.  **Local `.env` File:** A file named `.env` in the project root containing the necessary secrets. This file **must not** be committed to version control.

    **`.env` Template:**

    ```
    # Supabase connection string for session history
    SUPABASE_DB_URI=postgresql://...

    # GHL Private Integration Token
    GHL_API_TOKEN=pit-...

    # GHL Location ID (Sub-account ID)
    GHL_LOCATION_ID=...
    ```

---

## Deployment Steps

The deployment is a multi-step process. Steps 1 and 2 are typically a one-time setup. Step 4 is run for every deployment or update.

### Step 1: Grant Permissions (One-Time Setup)

The Cloud Run service needs an identity with permissions to access other Google Cloud services (like Vertex AI and Cloud Storage). We use a dedicated script to grant these roles to our existing service account.

**Script: `grant_permissions.sh`**

```bash
#!/bin/bash
set -e

# --- Configuration ---
SERVICE_ACCOUNT_EMAIL="stark-vertex-ai@ninth-potion-455712-g9.iam.gserviceaccount.com"
PROJECT_ID="ninth-potion-455712-g9"

echo "🛡️ Granting permissions to service account: $SERVICE_ACCOUNT_EMAIL"
echo "--------------------------------------------------"

# Grant permission to use Vertex AI
echo "1. Granting Vertex AI User role (roles/aiplatform.user)..."
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$SERVICE_ACCOUNT_EMAIL" \
  --role="roles/aiplatform.user"

# Grant permission to read from Google Cloud Storage
echo "2. Granting Storage Object Viewer role (roles/storage.objectViewer)..."
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$SERVICE_ACCOUNT_EMAIL" \
  --role="roles/storage.objectViewer"

echo "--------------------------------------------------"
echo "✅ All permissions granted successfully."
```
````

**To Use:** Make the script executable (`chmod +x grant_permissions.sh`) and run it once (`./grant_permissions.sh`).

---

### Step 2: Store Secrets (One-Time or When Secrets Change)

We do not hardcode secrets. This script reads your local `.env` file and securely stores the values in Google Secret Manager.

**Script: `store_secrets.sh`**

```bash
#!/bin/bash
set -e

# --- Configuration ---
PROJECT_ID="ninth-potion-455712-g9"
ENV_FILE=".env"

SECRETS_MAP=(
  "SUPABASE_DB_URI/adk-db-uri"
  "GHL_API_TOKEN/ghl-api-key"
  "GHL_LOCATION_ID/ghl-location-id"
)

echo "🔐 Storing secrets in Google Secret Manager for project: $PROJECT_ID"
echo "--------------------------------------------------"

if [ ! -f "$ENV_FILE" ]; then
    echo "❌ Error: $ENV_FILE not found!"
    exit 1
fi

for mapping in "${SECRETS_MAP[@]}"; do
  ENV_VAR_NAME="${mapping%%/*}"
  SECRET_NAME="${mapping##*/}"

  VALUE=$(grep -v '^#' "$ENV_FILE" | grep "$ENV_VAR_NAME" | cut -d '=' -f2-)

  if [ -z "$VALUE" ]; then
    echo "⚠️ Warning: Variable '$ENV_VAR_NAME' not found in $ENV_FILE. Skipping."
    continue
  fi

  echo "Storing '$ENV_VAR_NAME' as secret '$SECRET_NAME'..."
  if ! gcloud secrets describe "$SECRET_NAME" --project="$PROJECT_ID" &>/dev/null; then
    echo "  -> Secret '$SECRET_NAME' does not exist. Creating it now..."
    gcloud secrets create "$SECRET_NAME" --replication-policy="automatic" --project="$PROJECT_ID"
  fi

  echo -n "$VALUE" | gcloud secrets versions add "$SECRET_NAME" --data-file=- --project="$PROJECT_ID"
  echo "  -> ✅ Success."
done

echo "--------------------------------------------------"
echo "All secrets have been stored successfully."
```

**To Use:** Run `./store_secrets.sh` whenever you need to initialize or update your secrets in the cloud.

---

### Step 3: Core Build & Ignore Files

These files are essential for a correct and secure build process.

**File: `Dockerfile`**
This file defines how to build the container image. It installs dependencies into a clean environment and sets the startup command.

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD adk api_server . --host=0.0.0.0 --port=${PORT} --session_service_uri=${DB_URI}
```

**File: `.gcloudignore`**
This file is **critical**. It prevents local development folders (like `.venv`) and secret files from being uploaded, ensuring a clean, secure, and reproducible build.

```
# Ignore the virtual environment
.venv/

# Ignore Python cache files
__pycache__/
*.pyc

# Ignore Git directory and local environment files
.git/
.gitignore
.env

# Ignore any local log files or sensitive keys
*.log
*.json
```

---

### Step 4: Deploy to Cloud Run

This is the final script that brings everything together and deploys the service.

**Script: `deploy.sh`**

```bash
#!/bin/bash
set -e

# --- Configuration ---
SERVICE_NAME="adk-bundle-prod-v2"
REGION="us-east1"
PROJECT_ID="ninth-potion-455712-g9"
GCS_BUCKET_NAME="your-agent-instructions-bucket-name" # IMPORTANT: Set your bucket name
SERVICE_ACCOUNT_EMAIL="stark-vertex-ai@${PROJECT_ID}.iam.gserviceaccount.com"

echo "🚀 Deploying service: $SERVICE_NAME with explicit Vertex AI config..."

gcloud run deploy "$SERVICE_NAME" \
  --source . \
  --region="$REGION" \
  --project="$PROJECT_ID" \
  --allow-unauthenticated \
  --service-account="$SERVICE_ACCOUNT_EMAIL" \
  --set-secrets="DB_URI=adk-db-uri:latest,GHL_API_TOKEN=ghl-api-key:latest,GHL_LOCATION_ID=ghl-location-id:latest" \
  --set-env-vars="GOOGLE_CLOUD_PROJECT=${PROJECT_ID},GCS_BUCKET_NAME=${GCS_BUCKET_NAME},GOOGLE_CLOUD_LOCATION=${REGION},GOOGLE_GENAI_USE_VERTEXAI=TRUE"

echo "--------------------------------------------------"
echo "✅ Deployment complete!"
echo "📡 Service URL:"
gcloud run services describe "$SERVICE_NAME" --region="$REGION" --project="$PROJECT_ID" --format="value(status.url)"
```

**To Use:** Make the script executable (`chmod +x deploy.sh`) and run `./deploy.sh` to deploy or update your service.

```

```
