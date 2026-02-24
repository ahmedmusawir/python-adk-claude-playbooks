# ADK Agent Bundle Overview

This project is a multi-agent bundle developed using Google's Agent Development Kit (ADK). It is deployed as a secure, scalable service on **Google Cloud Run** and uses **Supabase (Postgres)** for session history management.

The bundle is designed to be the core "brain" of a larger system, exposing several specialized AI agents that can be accessed via a central **ADK Wrapper**.

---

## Agents Included

This bundle contains **five** distinct agents, each with a specific function:

### 1. `jarvis_agent`

- **Description:** A general-purpose conversational agent.
- **Tools:** Utilizes **Google Search** to answer questions about current events or general knowledge.

### 2. `calc_agent`

- **Description:** A specialized agent for performing mathematical calculations.
- **Tools:** Uses a custom `calculate` function to safely evaluate mathematical expressions.

### 3. `ghl_mcp_agent` ("Official GHL MCP - worthless!")

- **Description:** A CRM-integrated agent designed to interact with the **GoHighLevel (GHL)** platform. It can perform operations like searching for contacts, updating records, and viewing conversations.
- **Tools:** Connects to GHL's live data via the Model Context Protocol (MCP). This is the worthless official mcp server hosted by GHL

### 4. `greeting_agent` ("Rico")

- **Description:** An agent responsible for initial greetings and providing introductory information.
- **Tools:** Can read documents from **Google Cloud Storage (GCS)** to fetch context, such as a resume or a company profile. I reads from the out dated Resume of the Moose

### 5. `product_agent` ("Rico")

- **Description:** A product specialist agent that can answer detailed questions about specific products of **Dockbloxx.com**
- **Tools:** Reads from **Google Cloud Storage (GCS)** to fetch its knowledge base and context, ensuring its information is always up-to-date.

---
