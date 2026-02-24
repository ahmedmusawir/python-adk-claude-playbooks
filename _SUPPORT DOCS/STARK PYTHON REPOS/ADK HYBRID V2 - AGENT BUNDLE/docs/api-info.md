# API Information & Interaction Guide

This document provides the necessary information for interacting with the deployed ADK Agent Bundle.

---

## Service Endpoint

The ADK Agent Bundle is deployed as a Google Cloud Run service.

- **Live URL:** `https://adk-bundle-prod-v2-952978338090.us-east1.run.app`

---

## ⚠️ Important: Interaction Model

This ADK Agent Bundle service is a **backend component** and is **not designed to be called directly** by frontend applications or clients like Postman or n8n.

The native ADK API is complex, requiring specific, multi-step calls to create and manage user sessions before an agent can be run. Direct interaction is prone to error.

All requests **must** be routed through the **ADK Wrapper** service.

---

## The Role of the ADK Wrapper

The ADK Wrapper is a lightweight FastAPI application that acts as a simplified gateway to the ADK Agent Bundle. Its purpose is to:

1.  **Provide a Simple Endpoint:** It exposes a single, easy-to-use endpoint (e.g., `/run_agent`).
2.  **Abstract Complexity:** It automatically handles the creation and management of user sessions with the ADK service.
3.  **Route Requests:** It forwards the user's message to the correct agent on the live ADK bundle using the proper API format.

---

## Example API Call (to the Wrapper)

To interact with an agent, your client application (Streamlit, n8n, etc.) should make a `POST` request to the **wrapper's URL**, not the bundle's URL.

Here is an example using `curl`:

```bash
# The URL of your deployed ADK Wrapper service
WRAPPER_URL="[https://your-adk-wrapper-url.run.app](https://your-adk-wrapper-url.run.app)"

curl -X POST "${WRAPPER_URL}/run_agent" \
  -H "Content-Type: application/json" \
  -d '{
    "agent_name": "jarvis_agent",
    "message": "What is the latest news on Stark Industries stock?",
    "user_id": "tony_stark_001"
  }'
```
