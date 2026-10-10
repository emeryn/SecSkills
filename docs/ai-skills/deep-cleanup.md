---
name: deep-cleanup
description: Performs a deep cleaning of the project repository by removing debug comments, test/example folders, secrets, and personal data. Enforces .gitignore/.dockerignore checks, prevents AI skill leakage, scrubs hardcoded IPs, and forbids autonomous GitHub pushes.
---
# Deep Cleanup

This skill forces the AI agent to rigorously sanitize the project repository before a production release, ensuring no debug artifacts, mock data, or sensitive information remain in the codebase.

## When to Use
- The user asks to "clean the project", "prepare for production", or "remove test data".
- The user requests a review of secrets, PII, or debug comments.
- The project has passed human validation and is entering the final deployment phase.

## Core Rules & Cleanup Operations

### 1. Code & Comment Sanitization
- **Remove Debug Artifacts:** Systematically scan and remove all debug prints (e.g., `console.log`, `print()`), temporary comments (e.g., `// TODO: remove this`, `// test`), and dead/commented-out code.
- **Maintain JSDoc/Docstrings:** Do NOT remove architectural documentation, JSDoc, or Python docstrings. Only remove futile, procedural, or debug-oriented comments.

### 2. File & Directory Pruning
- **Delete Mock Data:** Identify and delete all temporary folders such as `example/`, `mocks/`, `dummy_data/`, or `test_assets/` that were used by the LLM or developer for scaffolding. (Do NOT delete legitimate unit test suites like `tests/` or `spec/` unless explicitly told to do so).
- **Wipe SQLite/Local DBs:** Remove any local `.sqlite`, `.db`, or temporary JSON database files that might contain test data.

### 3. Secret, PII & Network Eradication (Zero-Leak Policy)
- **Hardcoded Secrets:** Actively hunt for hardcoded API keys, passwords, JWT secrets, and tokens. Replace them entirely with environment variable references (e.g., `process.env.SECRET` or `os.environ.get('SECRET')`).
- **PII Scrubbing:** Ensure no Personally Identifiable Information (PII) such as real names, emails, phone numbers, or IP addresses were left in the codebase, tests, or seed files.
- **Network & IP Scrubbing:** You MUST check `docker-compose.yml`, configuration files, and scripts to ensure no specific residential, local LAN (e.g., `192.168.x.x`), or personal public IP addresses are hardcoded. Replace them with standard local loopbacks (`127.0.0.1`), generic placeholders, or environment variables.

### 4. Git & Docker Leak Prevention (Mandatory Review)
- **Review `.gitignore`:** You MUST read and verify the `.gitignore` file. Ensure it robustly ignores `.env`, `.env.*`, `logs/`, `*.pem`, `*.key`, IDE folders, local databases, AND all AI context/skill files (e.g., `.clauderules`, `.clinerules`, `.roomodes`, and the `docs/ai-skills/` directory).
- **Review `.dockerignore`:** You MUST read and verify the `.dockerignore` file. Ensure it strictly prevents secrets, local environment files, and AI context files from being copied into the Docker image build context.
- **AI Context Isolation (No Skill Leakage):** You MUST NEVER stage (`git add`), commit, or push the skill files or context rules used to guide your own behavior. They must remain strictly local to the development environment.

### 5. Strict Execution Boundaries (No Autonomous Push)
- **Never Push:** You MUST NEVER execute `git push` or any remote synchronization command autonomously.
- **Human Gatekeeper:** You may run `git status`, `git diff`, `git add`, and `git commit` to prepare the clean state, but the final action of pushing to a remote repository (like GitHub/GitLab) MUST be explicitly left to the human user.
- **Report Summary:** After the cleanup is complete, you MUST provide a concise list to the user detailing exactly what was deleted, what secrets were neutralized, and confirm the status of the ignore files.