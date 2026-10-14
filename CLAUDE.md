# Project Rules

Skills live in `.claude/skills/<name>/SKILL.md`. Before any task, load every skill whose domain matches the request and combine their rules.

| Domain | Skill |
|---|---|
| Security, architecture, auth, containers, scaffolding | `secure-app-scaffolder` |
| Local LLM, chatbots, agents, RAG, MCP | `offline-llm-orchestrator` |
| Pre-production, cleanup, secrets, PII | `deep-cleanup` |
| Project structure, README, code style, frontend layout & UX | `project-blueprint` |

`project-blueprint` applies to every task that creates or edits code, writes a README or touches the UI.

## Global rules
- **Language:** everything produced is in English (code, identifiers, comments, commit messages, README, docs, UI strings, logs, error messages) unless the user explicitly asks for another language. Replying to the user in their own language is fine; the project itself stays in English.
- **No change-log comments:** when fixing or modifying code, never add comments that describe the change, the fix or the request (`// fixed`, `// changed to X`, `# now uses Y`, `# added as requested`, `// FIX:`, `// updated`). Comments describe the code as it is, never its history. Remove such comments if you find them. History belongs in commit messages.
- **Minimal diffs:** a fix changes only what the fix requires. No unrelated reformatting, renaming or "improvements".

## Precedence
Security (`secure-app-scaffolder`, `offline-llm-orchestrator`) > `deep-cleanup` > `project-blueprint`. Report any conflict to the user instead of silently choosing.

## Hard limits
- Never stage, commit or push `CLAUDE.md` or `.claude/`.
- Never run `git push`. Run `git add` / `git commit` only when explicitly asked.
