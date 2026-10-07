---
name: offline-llm-orchestrator
description: Scaffolds and configures offline-first LLM chatbots and AI agents for air-gapped environments. Covers Ollama and OpenAI-compatible connectors, session segregation, custom root CA injection, proxy/secret management, optional agent mode, and full MCP (Model Context Protocol) server integration.
---
# Offline LLM Orchestrator

This skill guides the scaffolding and implementation of privacy-first, fully offline chatbots and AI agents inside isolated or air-gapped application stacks.

## When to Use
- The user wants to integrate local LLMs or AI agents into an application running offline or on a private network.
- The user asks for an Ollama connector or an OpenAI-compatible API client with air-gap hardening.
- The user needs isolated session management, custom internal PKI root CA support, prompt configuration interfaces for local models, or MCP (Model Context Protocol) server integration.

## Core Rules & Architecture

### 1. Dual Connectors (Offline-First)
- **Ollama Connector:** Native support for local Ollama endpoints (e.g., `/api/tags`, `/api/chat`, `/api/generate`).
- **OpenAI-Compatible Connector:** Generic endpoint support for any server mimicking the OpenAI schema (vLLM, LocalAI, text-generation-webui, llama.cpp server).
- **Model Discovery:** Upon connecting to a server endpoint, automatically fetch and list available models for selection.
- **Base Parameters:** Provide UI/API configuration for key inference parameters:
  - Temperature
  - Top_p
  - Max tokens / Context window
  - Frequency / Presence penalty

### 2. Network Isolation, PKI & Proxy
- **Internal Root CA Injection:** Provide an explicit mechanism to inject and trust private/enterprise Root CA certificates (`.pem` / `.crt`) to ensure TLS connections to local inference servers succeed without disabling verification (`verify=False` is strictly forbidden).
- **Proxy & Custom Headers:** Support proxy configuration (HTTP/HTTPS) and arbitrary custom headers per endpoint.
- **Secure Secret Storage:** API keys or bearer tokens must NEVER be stored in plaintext. Store secrets using application-level encryption (AES-256-GCM / Argon2 key derivation) or encrypted vaults.

### 3. Data Segregation & Sessions
- **Default Strict Isolation:** Every chat session belongs to a single authenticated user. Enforce access control checks on every session read/write.
- **Shared Agent Exception:** Data sharing across users is permitted ONLY when the user explicitly opts into a shared workspace or common project context.

### 4. On-Demand Menus & Agent Mode
- **No Unsolicited Agent Mode:** Do NOT implement autonomous agents, multi-step tool calls, or background loops unless explicitly requested by the user. Default to standard conversational chat.
- **Optional UI/Menus:** LLM configuration menus and agent controls must remain optional or hidden until explicitly activated.

### 5. System Prompt & Fallback Controls
- **Customizable System Prompts:** If an agent or chat requires a system/base prompt to operate, expose a dedicated configuration section allowing the user to inspect and edit it.
- **Factory Reset:** ALWAYS implement a "Reset to Default" action to restore original system prompts without data loss.

### 6. Model Context Protocol (MCP) Integration
- **MCP Server Capability:** The user MUST be able to request the service to act as an MCP Server. This feature must be toggleable from the LLM configuration menu.
- **Standard MCP Endpoint:** When activated, the MCP server must natively expose all functions/tools of the service. A user or service account with a valid API key must be able to exploit these exposed functions seamlessly.
- **Admin MCP Endpoint:** The administrator MUST have the ability to activate a dedicated Admin MCP endpoint that allows end-to-end configuration of the service. This endpoint MUST strictly require an Admin API Key.
- **MCP Audit Logging:** All MCP actions (tool calls, resource access) MUST be heavily logged in a structured format, making it easy for the administrator to analyze automated interactions.

## Implementation Checklist
- [ ] TLS verification uses custom internal CA bundle when provided.
- [ ] Connectors support streaming responses (SSE).
- [ ] Tokens and keys are encrypted at rest.
- [ ] User ID is enforced in database queries for conversation history.
- [ ] Model list is queried on demand via `/v1/models` or `/api/tags`.
- [ ] MCP Server integration is toggleable and secured by API keys.
- [ ] Admin MCP is isolated and requires explicit admin credentials.
- [ ] Full logging for all MCP transactions is implemented.