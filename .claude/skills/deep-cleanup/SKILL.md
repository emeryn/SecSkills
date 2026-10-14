---
name: deep-cleanup
description: Use when preparing a repository for production or release, or when asked to clean the project or remove test/mock data, debug code, AI change-log comments, secrets or personal data. Runs an inventory-first cleanup (debug artifacts, mock folders, local DBs, hardcoded secrets, PII, file metadata, LAN/personal IPs, debug flags), verifies .gitignore/.dockerignore and image layers, scans working tree and git history, keeps AI context files out of git and never pushes.
---
# Deep Cleanup

Sanitizes a repository before production: no debug artifact, mock data, secret or personal data in the code, the image or the history.

## When to Use
- "Clean the project", "prepare for production", "remove test data", "check for secrets/PII".
- After human validation, before the first deployment or a release.

## Process (always in this order)
1. **Inventory (read-only):** list everything that would be changed or deleted, grouped by the categories below.
2. **Confirm:** deleting anything not obviously disposable (ambiguous folders, data files, scripts) requires user confirmation. Debug prints, commented-out code and AI change-log comments can be removed directly.
3. **Apply.**
4. **Verify:** build, lint and tests pass; secret scanners return no findings.
5. **Report** (last section).

## 1. Code & Comments
- Remove debug output (`console.log`, `print()`, `debugger`, `pdb`, `breakpoint()`), commented-out code and temporary comments (`// test`, `# TODO remove`).
- **Remove AI/change-log comments:** comments describing a modification or its reason rather than the code (`// fixed`, `// FIX:`, `# changed from X to Y`, `// updated per request`, `# now uses`, `// added by AI`, `// new`, `// previously`). Rewrite as a neutral *why* comment only if the reason is still useful to a reader.
- `TODO`/`FIXME` replaced by a ticket reference or deleted.
- Keep docstrings, JSDoc and comments explaining *why*. Remove comments that paraphrase the code.
- Non-English comments, identifiers, UI strings or docs are translated to English unless the user asked for another language.
- Logging through the project logger stays; only ad-hoc debug output goes.

## 2. Files & Directories
- Delete scaffolding leftovers: `example/`, `examples/` (unless documented as user-facing), `mocks/`, `dummy_data/`, `test_assets/`, `tmp/`, `*.bak`, `*_old.*`, `copy of *`, scratch files.
- Delete local databases and dumps: `*.sqlite`, `*.db`, `*.dump`, data-bearing `*.sql`, ad-hoc JSON stores.
- Clear Jupyter notebook outputs (`nbstripout`) or remove scratch notebooks.
- **Never delete:** real test suites (`tests/`, `spec/`, `e2e/`), migrations, lockfiles, `.env.example`, seccomp profiles, bootstrap admin logic, `docs/help/`, `docs/security/`, `docs/screenshots/`, `frontend/e2e/fixtures/` (screenshot demo data).
- Remove empty directories and unused dependencies (`depcheck`, `deptry`).

## 3. Secrets, PII & Network
- **Secrets:** API keys, passwords, JWT secrets, private keys, connection strings, tokens in `.npmrc`, `.pypirc`, `pip.conf`, `.docker/config.json`, `kubeconfig`, CI files. Replace with environment/secrets-provider references and add the variable to `.env.example` with a placeholder.
- **Secrets in git history:** `gitleaks detect --source . --log-opts="--all"` (or `trufflehog git file://.`). Any secret found in history is compromised: report it and require rotation. Do NOT rewrite history; recommend `git filter-repo` to the user.
- **PII:** no real names, emails, phones, addresses or IPs in code, fixtures, seeds, screenshots, logs or docs. Use `example.com`, `+33 1 00 00 00 00`, RFC 5737 IPs (`192.0.2.0/24`).
- **Screenshots:** check `docs/screenshots/` for real names, emails, hostnames, IPs or internal URLs; regenerate with `make screenshots` if needed.
- **File metadata:** strip EXIF from images (`exiftool -all=`) and author/company metadata from PDF/Office files.
- **Network:** no LAN (`192.168.x.x`, `10.x`, `172.16-31.x`), personal public IPs, internal hostnames, internal registry URLs or usernames in compose, configs, lockfiles, scripts or README. Use `127.0.0.1`, service names or environment variables.
- **Git metadata:** report (do not change) personal emails used as commit authors if the repository will be published.

## 4. Production Settings
- Debug off (`DEBUG=False`, `NODE_ENV=production`, no `--reload`), dev dependencies absent from runtime images.
- No source maps served, OpenAPI UI disabled or admin-only, no verbose error pages.
- No default credentials; bootstrap admin uses a generated password + forced reset.
- CORS allowlist contains no `localhost` entries in production config.
- README checked against `project-blueprint`: no machine-specific paths, no outdated commands, scan dates up to date.

## 5. Ignore Files & Images (mandatory review)
- **`.gitignore`** covers at least: `.env`, `.env.*` (with `!.env.example`), `*.pem`, `*.key`, `*.p12`, `*.pfx`, `logs/`, `*.sqlite`, `*.db`, `node_modules/`, `.venv/`, `__pycache__/`, `dist/`, `build/`, `coverage/`, IDE folders, and AI context files (`CLAUDE.md`, `CLAUDE.local.md`, `.claude/`, `.clauderules`, `.clinerules`, `.roomodes`, `.cursor/`, `AGENTS.md`).
- **`.dockerignore`** additionally excludes `.git`, tests, docs, `node_modules`, local envs and AI context files.
- **Built images:** `trivy image --scanners secret` and `docker history --no-trunc` show no secret, `.env` or AI context file in any layer.

## 6. Git Boundaries
- Never stage, commit or push AI context files.
- Never run `git push` or any remote sync command.
- `git add`/`git commit` only when the user explicitly asks; otherwise leave changes unstaged for review.
- Never rewrite history (`rebase`, `reset --hard`, `filter-repo`) autonomously.

## Report
- Deleted files/folders (paths).
- Debug code and AI change-log comments removed (count per file).
- Secrets neutralized (file + variable name, never the value) and history secrets requiring rotation.
- PII, metadata and IP replacements.
- `.gitignore` / `.dockerignore` changes, image layer check result.
- Verification results (build, tests, gitleaks) and open decisions for the user.
