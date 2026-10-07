# 🛡️ AI Skills Library for Secure & Offline Architecture

This repository contains a collection of custom AI Skills designed to force AI assistants (like Claude, Cline, or Roo Code) to generate **secure-by-design** and **offline-first** application boilerplates.

By feeding these skills to your AI, it will stop generating generic, insecure code and instead output production-ready architectures featuring rootless execution, SBOM generation, post-quantum cryptography readiness, strict identity management, and air-gapped LLM/MCP integrations.

## 📂 Project Structure

To manage multiple skills efficiently without confusing the AI, we use a "Master Rule" approach. Place the skill files in a dedicated documentation folder:

```text
your-project/
├── .clauderules                 <-- The Master Rule file read by the AI
├── docs/
│   └── ai-skills/
│       ├── secure-app-scaffolder.md
│       └── offline-llm-orchestrator.md
├── src/
└── ...