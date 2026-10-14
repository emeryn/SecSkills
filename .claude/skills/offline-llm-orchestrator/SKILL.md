---
name: offline-llm-orchestrator
description: Use when integrating a local LLM, chatbot, RAG search or AI agent into an application running offline, air-gapped or on a private network. Covers Ollama and OpenAI-compatible connectors (vLLM, llama.cpp, LocalAI), model discovery and integrity, streaming, internal root CA, proxy and encrypted secrets, per-user session and RAG isolation, prompt-injection and exfiltration defenses, editable system prompts with reset, opt-in agent mode and exposing the service as an MCP server.
---
# Offline LLM Orchestrator

Scaffolds privacy-first chatbots, RAG and AI agents running entirely on local or private inference servers.

Combine with `secure-app-scaffolder` (auth, authorization, logging, SSRF, uploads, MCP endpoints) and `project-blueprint` (structure, UI). Security rules take precedence.

## When to Use
- Adding a chatbot, assistant, summarizer, document Q&A (RAG) or agent backed by a local model.
- Building an Ollama or OpenAI-compatible client for a private network.
- Configuring TLS with an internal PKI, proxies, model parameters, system prompts or MCP.

## 1. Connectors
- **One interface, two drivers:** `LLMProvider` (`list_models`, `chat`, `chat_stream`, `embed`, `health`):
  - **Ollama:** `/api/tags`, `/api/chat`, `/api/show`, `/api/embed`.
  - **OpenAI-compatible:** `/v1/models`, `/v1/chat/completions`, `/v1/embeddings` (vLLM, llama.cpp server, LocalAI, TGI).
- **Model discovery:** on connection and on demand; never a single hardcoded model.
- **Parameters (per endpoint, overridable per conversation):** temperature, top_p, max tokens, context window, frequency/presence penalty, stop sequences. Ranges validated server-side; max tokens capped by an admin limit.
- **Streaming:** SSE to the browser; `proxy_buffering off;` behind Nginx; cancellation closes the upstream request.
- **Resilience:** connect/read timeouts, retries only before the first token, clear "server unavailable" state in the UI.
- **Context management:** count tokens before sending; truncate or summarize oldest messages instead of failing.

## 2. Inference Server Exposure
- Ollama and most local servers have **no authentication**: never publish their port. They live on an internal network reachable only by the backend; the browser never calls them directly.
- If the server is remote, put it behind a reverse proxy with TLS and a bearer token or mTLS.

## 3. Models
- Only `safetensors` or `GGUF` formats; never pickle-based files (`.bin`, `.pt`, `.pkl`) or `trust_remote_code=True`.
- Models are imported from an internal repository with a SHA-256 checksum verified before loading; the list of approved models and checksums is kept in `deploy/models.lock` (or equivalent).

## 4. Network, PKI & Secrets
- **Internal Root CA:** admins upload a `.pem`/`.crt` bundle; the HTTP client uses it via an explicit `SSLContext`/`verify=<bundle_path>`. Disabling verification is forbidden.
- **Proxy & headers:** optional HTTP(S) proxy, `NO_PROXY` list, custom headers per endpoint.
- **SSRF:** endpoint URLs are admin-only settings validated as in `secure-app-scaffolder` (private ranges allowed only through an explicit admin allowlist).
- **Outbound credentials:** encrypted at rest (AES-256-GCM), never returned to the browser (`••••last4`).
- **No secrets in prompts:** system prompts and tool descriptions never contain credentials, internal URLs or data the user is not allowed to see; assume the model can be made to reveal them.
- **Truly offline:** no telemetry or hub calls (`HF_HUB_OFFLINE=1`, `TRANSFORMERS_OFFLINE=1`, `OLLAMA_NOHISTORY=1`). Tokenizers and embedding models loaded from local paths.

## 5. Sessions, RAG & Data Isolation
- **Ownership:** each conversation belongs to one user; `user_id` filtered in the repository layer; cross-user access returns `404`.
- **Shared workspaces:** only when the user explicitly creates or joins one; access checked against membership.
- **RAG authorization:** document chunks carry owner/workspace/ACL metadata; retrieval filters on the requesting user's permissions **before** similarity search results reach the prompt. One collection (or mandatory filter) per tenant.
- **Uploads for RAG** follow the upload rules of `secure-app-scaffolder` (size, type, parsing in a sandboxed worker with limits on pages and extracted text).
- **Retention:** configurable retention and purge job; deleting a conversation or document also deletes its embeddings.
- **Logging:** prompts/completions not logged by default; when enabled, PII masking applies.

## 6. Prompt Injection & Output Safety
- Model output is untrusted: Markdown rendered through a sanitizer, no raw HTML, no auto-loaded remote images (blocks data exfiltration via image URLs), links shown with their real target and `rel="noopener noreferrer"`.
- Retrieved documents, tool results and web content are passed as delimited data, never merged into system instructions; they cannot change tools or permissions.
- Model-generated code, SQL or shell is never executed automatically.
- Tool calls proposed by the model are validated against a schema and the **user's** RBAC (the model never has more rights than the user), with size limits on tool outputs.
- Rate limits and per-user token quotas on generation endpoints.

## 7. UI: Chat, Configuration & Agent Mode
- **Default:** plain chat in the `project-blueprint` shell: conversation list left, messages center, input bottom. Stop, regenerate, copy, Markdown/code rendering, sources shown for RAG answers.
- **Configuration hidden by default:** model and parameter settings in a collapsible panel or the admin area.
- **No unsolicited agent mode:** no autonomous loops, multi-step tool calls or background jobs unless explicitly requested. When requested: explicit toggle, visible step trace, max steps and timeout, human confirmation before any destructive or external action.

## 8. System Prompts
- Every system prompt is visible and editable in a settings section, versioned (who/when).
- **Reset to default** restores the factory prompt without deleting conversations.
- Default prompts live in `backend/app/llm/prompts/*.md`, not inline in code.

## 9. MCP
- **Server side:** implement the User and Admin MCP endpoints exactly as defined in `secure-app-scaffolder` §9. The toggle lives in the LLM configuration menu.
- **Client side (agent mode only):** external MCP servers from an admin-managed allowlist, per-tool enable/disable, tool descriptions pinned (a server changing its tool definitions requires admin re-approval), user confirmation for write actions.

## Implementation Checklist
- [ ] Inference server port not published; backend is the only client.
- [ ] Models in safetensors/GGUF with verified checksums.
- [ ] TLS uses the internal CA bundle; no verification bypass.
- [ ] SSE streaming works through the proxy and supports cancel.
- [ ] Outbound credentials encrypted and never sent to the browser.
- [ ] `user_id`/ACL enforced in conversation queries and RAG retrieval.
- [ ] Model output sanitized; no remote images.
- [ ] Agent mode and MCP off by default; every MCP call audited.

## Gotchas
- Nginx buffering silently breaks streaming: the answer arrives all at once.
- Ollama and OpenAI-compatible servers return different error shapes; normalize them in the provider.
- Some servers ignore unsupported parameters silently; show effective parameters in the UI.
- Filtering RAG results after the top-k search leaks nothing but returns empty answers; filter inside the vector query.
- Retrying a streamed request duplicates output; retry only before the first token.
