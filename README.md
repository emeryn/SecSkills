# 🛡️ Secure App & Offline AI Architecture

This repository uses AI Skills to enforce a **secure-by-design** and **offline-first** application boilerplate. It requires AI assistants (like Claude, Cline, or Roo) to adhere strictly to ANSSI/CIS/FBI guidelines and isolated execution environments.

## 📂 Project Structure & AI Skills

This project is guided by dynamic rules located in the documentation folder:

- `.clauderules` (Root) : Forces the AI to read the skills before executing tasks.
- `docs/ai-skills/secure-app-scaffolder.md` : Enforces zero-trust input, rootless containers, idempotency, column-level encryption, and active defense.
- `docs/ai-skills/offline-llm-orchestrator.md` : Enforces offline-first AI integration, strict session segregation, PKI/Root CA injection, and secure MCP server deployment.
- `docs/ai-skills/deep-cleanup.md` : Enforces rigorous pre-production sanitization (PII scrubbing, secret removal, residential IP removal, and AI context isolation).
