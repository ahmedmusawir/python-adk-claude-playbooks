# Multi-Runtime Docker

## What This Is For

Run Python ADK agents + TypeScript MCP servers in a single Cloud Run container. Nginx as the public port, Supervisor managing multiple processes, SSE streaming configured correctly.

Sourced from the GHL Agent + MCP Server repo — production deployment on Cloud Run.

---

## Quick Reference — Port Layout

```
PUBLIC
  Nginx :8080
    /          → ADK api_server :8000  (SSE streaming)
    /mcp       → MCP TypeScript server :9000
    /health    → returns 200 directly
INTERNAL
  ADK api_server :8000
  MCP server :9000
```

Nginx is the only port exposed to Cloud Run's public traffic. ADK and MCP run internally.

---

## Complete Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# ─── System Dependencies ───────────────────────────────────────────────────
RUN apt-get update && apt-get install -y \
    nodejs \
    npm \
    nginx \
    supervisor \
    apache2-utils \
    && rm -rf /var/lib/apt/lists/*

# ─── Python Dependencies ───────────────────────────────────────────────────
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ─── TypeScript MCP Server Build ───────────────────────────────────────────
COPY mcp-server/package*.json ./mcp-server/
RUN cd mcp-server && npm ci

COPY mcp-server/ ./mcp-server/
RUN cd mcp-server && npm run build

# ─── Nginx Config ─────────────────────────────────────────────────────────
COPY nginx.conf /etc/nginx/nginx.conf

# Optional: Basic Auth
# RUN htpasswd -bc /etc/nginx/.htpasswd admin your-password-here

# ─── Supervisor Config ─────────────────────────────────────────────────────
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# ─── Application Code ──────────────────────────────────────────────────────
COPY . .

# ─── Startup ───────────────────────────────────────────────────────────────
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

---

## supervisord.conf

```ini
[supervisord]
nodaemon=true       ; ← Required for Docker. Without it, supervisord daemonizes and the container exits.
logfile=/dev/null
pidfile=/tmp/supervisord.pid

[program:nginx]
command=/usr/sbin/nginx -g "daemon off;"
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0

[program:adk]
command=bash -c "adk api_server . --port 8000"
directory=/app
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0

[program:mcp]
command=node /app/mcp-server/dist/server.js
directory=/app/mcp-server
autostart=true
autorestart=true
environment=PORT=9000,GHL_API_KEY=%(ENV_GHL_API_KEY)s,GHL_LOCATION_ID=%(ENV_GHL_LOCATION_ID)s
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0
```

**nodaemon=true is critical.** Without it, supervisord runs as a background daemon and Docker's PID 1 exits immediately, killing the container.

**Environment passthrough:** Supervisor passes env vars to child processes via `%(ENV_VAR_NAME)s` syntax. List all env vars your MCP server needs in the `[program:mcp]` environment line.

---

## nginx.conf

```nginx
worker_processes 1;

events {
    worker_connections 1024;
}

http {
    # Required for SSE streaming
    upstream adk {
        server localhost:8000;
    }

    upstream mcp {
        server localhost:9000;
    }

    server {
        listen 8080;

        # ─── Health Check ──────────────────────────────────────────────────
        location /health {
            return 200 "ok\n";
            add_header Content-Type text/plain;
        }

        # ─── MCP Server ────────────────────────────────────────────────────
        location /mcp {
            proxy_pass http://mcp;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_read_timeout 60s;
        }

        # ─── ADK API Server (SSE Streaming) ────────────────────────────────
        location / {
            proxy_pass http://adk;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;

            # SSE streaming config — all five lines required
            proxy_buffering off;        # ← Don't buffer — stream immediately
            proxy_cache off;            # ← No caching for streaming
            proxy_read_timeout 300s;    # ← Long timeout for slow agent responses
            proxy_http_version 1.1;     # ← Required for chunked transfer
            proxy_set_header Connection "";  # ← Clears hop-by-hop header for keep-alive
        }
    }
}
```

**All five SSE config lines are required.** Missing any one causes streaming responses to buffer (user waits for entire response), timeout, or fail to stream chunks.

---

## nginx.conf with Basic Auth

Add a simple auth layer to protect the endpoint:

```nginx
# In Dockerfile: RUN htpasswd -bc /etc/nginx/.htpasswd admin your-password

location / {
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;

    proxy_pass http://adk;
    proxy_buffering off;
    proxy_cache off;
    proxy_read_timeout 300s;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
}
```

For production, use Secret Manager for the password and inject it at build time, or use Cloud Run's built-in IAM authentication instead.

---

## .gcloudignore for Multi-Runtime Projects

```
.git
.gitignore
.venv
__pycache__
*.pyc

# Node — don't upload, Docker builds it
mcp-server/node_modules/
mcp-server/dist/

# Local dev
.env
inspect_code/
```

**Important:** Exclude `mcp-server/node_modules/` — it can be hundreds of MB. Docker rebuilds it during `npm ci`. If you don't exclude it, Cloud Build upload takes forever.

---

## deploy.sh for Multi-Runtime

```bash
#!/bin/bash
PROJECT="your-project"
REGION="us-east1"
SERVICE="ghl-agent-mcp"

# Build and push
gcloud builds submit \
  --tag gcr.io/${PROJECT}/${SERVICE} \
  --project ${PROJECT}

# Deploy
gcloud run deploy ${SERVICE} \
  --image gcr.io/${PROJECT}/${SERVICE} \
  --region ${REGION} \
  --project ${PROJECT} \
  --allow-unauthenticated \
  # Nginx's listen port
  --port 8080 \
  --memory 2Gi \
  --cpu 2 \
  --min-instances 1 \
  --no-cpu-throttling \
  --cpu-boost \
  --timeout 300 \
  --set-env-vars="GOOGLE_GENAI_USE_VERTEXAI=1,GOOGLE_CLOUD_PROJECT=${PROJECT},GOOGLE_CLOUD_LOCATION=${REGION}" \
  --set-secrets="GHL_API_KEY=ghl-api-key:latest,GHL_LOCATION_ID=ghl-location-id:latest" \
  --service-account "agent-sa@${PROJECT}.iam.gserviceaccount.com"
```

---

## Startup Ordering

Supervisor starts all processes simultaneously by default. If ADK starts before the MCP server is ready, the first agent request may fail to connect to MCP tools.

Add a startup delay to the ADK process, or add a health check probe:

```ini
[program:adk]
command=bash -c "sleep 5 && adk api_server . --port 8000"
```

The 5-second delay lets the MCP server fully start and register its tools before ADK tries to connect. First seen in: GHL Agent + MCP Server repo.

---

## Python Requirements

```
google-adk>=0.1.0
google-cloud-secret-manager
google-cloud-storage
google-generativeai
```

**Don't** include Node.js or MCP SDK packages in `requirements.txt` — those are in `mcp-server/package.json`.

---

## Logs in Cloud Run

With the supervisord config above, all process logs go to stdout/stderr which Cloud Run captures automatically. View in Cloud Logging:

```bash
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=ghl-agent-mcp" \
  --limit 100 \
  --format="table(timestamp, textPayload)"
```

---

## Gotchas

**1. nodaemon=true is critical**
Without it, supervisord forks into the background and Docker's PID 1 (supervisord itself) exits, killing all child processes and the container. The container appears to start, then immediately exits.

**2. Nginx SSE buffering**
All five SSE lines are needed for ADK streaming: `proxy_buffering off`, `proxy_cache off`, `proxy_read_timeout 300s`, `proxy_http_version 1.1`, and `proxy_set_header Connection ""`. Missing `proxy_buffering off` causes the browser to wait for the complete response. Missing the timeout causes long agent runs to be cut off. Missing `proxy_http_version 1.1` breaks chunked transfer encoding.

**3. TypeScript compiled in Dockerfile**
Supervisor runs `node dist/server.js`. If you forget `npm run build` in the Dockerfile, the `dist/` directory doesn't exist and the MCP process crashes immediately.

**4. node_modules not in .gcloudignore**
Cloud Build uploads everything not in `.gcloudignore`. `node_modules/` can be 300MB+. Always exclude it. The Docker `npm ci` step rebuilds it inside the image.

**5. Environment variables for child processes**
Supervisor doesn't automatically pass all environment variables to child processes. List required env vars explicitly in the `[program:mcp]` `environment=` line using `%(ENV_VAR_NAME)s` syntax.

**6. Cloud Run port must match Nginx listen port**
Cloud Run sends traffic to the port specified in `--port`. Nginx must `listen` on that exact port. If you set `--port 8080`, Nginx must `listen 8080;` — not 80.

---

## What's Next

- Build the TypeScript MCP server → `mcp-server-typescript.md`
- Connect ADK agent to MCP tools → `adk-advanced-patterns.md`
- Deploy to Cloud Run → `cloud-run-deployment.md`
