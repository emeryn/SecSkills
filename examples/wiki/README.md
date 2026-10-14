# Example: Secure Wiki

Step-by-step kickoff of a complete project with Claude Code and the four skills. The target app is described in [specification.md](specification.md).

## 1. Setup
```bash
mkdir wiki && cd wiki && git init
cp -r <SecSkill>/CLAUDE.md <SecSkill>/.claude .
mkdir docs && cp <SecSkill>/examples/wiki/specification.md docs/
printf '\n## Product\nThe functional specification is `docs/specification.md`. Read it before any feature work.\n' >> CLAUDE.md
claude
```
Add `CLAUDE.md`, `.claude/` to `.gitignore` in phase 1 (the agent does it per `deep-cleanup`).

## 2. Working Method
- One phase per session: run `/clear` between phases, the context restarts from `CLAUDE.md` + skills + `docs/`.
- Start every phase in plan mode (`Shift+Tab`), review the plan, then let it implement.
- A phase is done only when its "Done when" checks pass on your machine. Commit yourself after review.
- Never paste secrets into the conversation.

## 3. Phases
Copy each prompt as is.

### Phase 1 — Plan
```text
Read docs/specification.md. Using the secure-app-scaffolder and project-blueprint skills, produce:
the directory tree, docs/security/threat-model.md, the data model (Mermaid ER diagram),
the authentication sequence diagrams, and a phased implementation plan matching the phases below.
Do not write application code yet. List every decision or ambiguity you need me to settle.
```
**Done when:** you have answered the open questions and `docs/security/threat-model.md` exists.

### Phase 2 — Foundation
```text
Implement the foundation: Makefile with all standard targets, docker-compose.yml based on the
reference template, Dockerfiles, secret bootstrap, Nginx config, backend app
factory with config/logging/db/health, Alembic setup, frontend AppShell with sidebar, top bar,
theme, i18n (en), error and empty states, .gitignore, .dockerignore, .env.example, pre-commit.
```
**Done when:** on a clean clone `docker compose up -d` works on the first try without `.env` or `make`, the same with `podman compose up -d` rootless, `https://localhost:8443` shows the empty shell, `make lint test` pass, `docker compose exec backend id` is not root.

### Phase 3 — Identity
```text
Implement local authentication (Argon2id, NIST password policy), server-side sessions,
CSRF, idempotency store, bootstrap admin with forced reset and MFA enrollment (TOTP + WebAuthn),
password reset, session list/revoke, RBAC with deny-by-default route permissions, users and
groups admin pages, audit trail with hash chaining. Add the authorization and auth tests from
reference/security-tests.md.
```
**Done when:** `make admin-password` + first login forces reset and MFA; the generated authorization tests cover every route.

### Phase 4 — Wiki Core
```text
Implement spaces, space members, pages (tree, move, trash), Markdown editor with preview,
internal [[links]], revisions with diff and restore, optimistic locking with If-Match/412,
recent changes and favorites, following docs/specification.md and its security requirements.
```
**Done when:** two browsers editing the same page get a 412 merge view; a page with `<script>`, `javascript:` links and remote images renders inert.

### Phase 5 — Search, Attachments, Export
```text
Implement PostgreSQL full-text search filtered by permissions in SQL, attachments with the upload
rules (magic bytes, re-encoding, size limits), Markdown zip export, with their security tests.
```
**Done when:** a user never sees, in search, exports or link resolution, the title of a page from a restricted space they are not member of.

### Phase 6 — Administration & Help
```text
Implement the admin area: dashboard (usage, errors, honeytoken alerts), settings for remote syslog,
SMTPS and webhooks with SSRF validation and "Send test", PII masking toggle, audit viewer with
filters and export, scheduled jobs (trash purge, inactive accounts, token cleanup, encrypted backups),
embedded help from docs/help for users and admins.
```
**Done when:** a webhook to `http://169.254.169.254/` is rejected; `make restore-test` restores a backup.

### Phase 7 — Optional Extensions
Pick only what you need, one per session:
```text
Add OIDC (code + PKCE) and LDAPS login with group-to-space-role mapping, configurable either
from the admin UI or from BACKEND_OIDC_* / BACKEND_LDAP_* environment variables (env wins and
is shown read-only in the UI), with a "Test connection" action.
```
```text
Using offline-llm-orchestrator, add "Ask the wiki": RAG over pages the user can read, with an
Ollama connector, sources displayed, and permission filtering inside the vector query.
```
```text
Expose the user and admin MCP endpoints with tools search_pages, read_page, create_page, update_page.
```

### Phase 8 — Hardening & README
```text
Run make scan and make sbom, fix findings, complete the security test matrix, run OWASP ZAP baseline,
then generate the screenshots with make screenshots and write README.md from the project-blueprint
template with the real screenshots, scan results and dates.
```
**Done when:** no critical/high CVE without written justification, README ≤ 150 lines with real screenshots, every README command works on a fresh clone.

### Phase 9 — Pre-production Cleanup
```text
Run the deep-cleanup skill. Show me the inventory before deleting anything.
```
**Done when:** the report shows no secret in tree, history or image layers, and build and tests pass.
