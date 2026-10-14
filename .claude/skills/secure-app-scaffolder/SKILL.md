---
name: secure-app-scaffolder
description: Use when starting, scaffolding or re-architecting a web application that must be secure-by-design (ANSSI/CIS/FBI/OWASP ASVS), or when adding auth, authorization, uploads, audit logging, admin features, Docker/Podman deployment or MCP endpoints. Produces a rootless TypeScript frontend + Python backend + hardened PostgreSQL stack with zero-trust input, object-level authorization, column-level encryption, idempotent writes, OIDC/LDAPS/WebAuthn, tamper-evident audit logs, honeytokens, signed images, dual scanning (Trivy + OpenSCAP) and a pre-production cleanup procedure.
---
# Secure App Scaffolder

Generates a secure-by-design application compliant with ANSSI, CIS, FBI and OWASP ASVS level 2: TypeScript frontend, Python backend, hardened PostgreSQL.

Layout, README and UI follow the `project-blueprint` skill. Security rules in this skill take precedence over any convenience or UX rule.

## Reference files (read when generating the matching artifact)
- [reference/docker-compose.yml](reference/docker-compose.yml): hardened compose with secret bootstrap.
- [reference/nginx.conf](reference/nginx.conf): reverse proxy with TLS 1.3, hybrid PQ, headers, rate limits.
- [reference/security-tests.md](reference/security-tests.md): mandatory security test matrix.

## When to Use
- Starting, scaffolding or architecting a new web application or service.
- Requests mentioning ANSSI, CIS, FBI, OWASP, hardening, rootless, compliance.
- Adding auth, authorization, file uploads, audit logging, admin features, deployment files or MCP endpoints.

**Deployment targets:** Docker and rootless Podman only. Do not generate Kubernetes, Helm or other orchestrator manifests unless the user explicitly asks.

## 0. Threat Model
- Create `docs/security/threat-model.md` (≤ 1 page): assets, trust boundaries, actors, top risks (STRIDE) and the control mitigating each. Update it when a new trust boundary appears (upload, webhook, LLM, MCP).

## 1. Containers & Runtime
- **Rootless:** every image ends with a numeric `USER` (e.g. `USER 10001:10001`), including the web server (`nginxinc/nginx-unprivileged`, port > 1024).
- **Minimal, pinned images:** multi-stage builds; runtime stage is distroless or slim with no shell, compiler or package manager; dev dependencies excluded. Base images pinned by digest. Latest stable Python.
- **Hardening:** `read_only`, `tmpfs` for writable paths, `cap_drop: [ALL]`, `no-new-privileges`, runtime default seccomp profile (never `unconfined`), `pids_limit`, memory limits, healthchecks, core dumps disabled (`ulimits: core: 0`). Only the reverse proxy publishes ports; the database network is `internal: true`. Never mount the Docker/Podman socket.
- **No sub-processes:** no `subprocess`, `os.system`, `shell=True`. Enforced by: no shell in the image, a SAST rule, and an in-process seccomp filter denying `execve`/`execveat` loaded in each worker after startup (a container-level deny would prevent the entrypoint itself from starting).
- **First-run guarantee:** on a clean clone, `docker compose up` (also `docker-compose up` and `podman compose up`) works on the first try: no `.env`, no manual step, no prerequisite other than the container engine. Images build automatically (`build:`), only Compose Specification features supported by Docker Compose v2 and podman-compose are used, no host paths outside the repository, published ports > 1024, no CPU limits (unavailable in many rootless cgroup setups). `make up` is a thin wrapper around the same command.
- **No hardcoded secrets:** A one-shot `init` service generates secrets into a named volume (`0600`) consumed through `*_FILE` variables. Secrets are read from files, not environment variables, in production.
- **Air-gapped:** all JS/CSS/fonts/icons bundled; no CDN, runtime download or telemetry. Dependencies come from an internal mirror (Nexus, Artifactory, devpi, Verdaccio) when building offline.
- **Vault-ready:** a single secrets provider abstraction (`file` default, `vault`/`sops` in production).
- **Internal TLS:** backend ↔ database over TLS; optional mTLS between services.

## 2. Supply Chain & Scanning
- **Locked dependencies:** commit lockfiles; `npm ci`, `uv sync --frozen` or `pip install --require-hashes`. `.npmrc` with `ignore-scripts=true` unless a package requires install scripts (allowlist).
- **Pre-build gate:** `npm audit --audit-level=high` and `pip-audit` fail the build on exit code > 0.
- **SAST & secrets:** pre-commit with `ruff`, `bandit`/`semgrep`, `eslint` (+ `eslint-plugin-security`), `gitleaks`.
- **Image scans (`make scan`):** `trivy image` (CVEs + secrets + misconfig), `trivy config` on Dockerfiles and compose, `oscap-podman` (or `oscap-docker`) against the CIS profile of the base OS.
- **SBOM & signing (`make sbom`):** CycloneDX SBOM (Trivy or Syft), images signed with `cosign` (key-based for offline), SBOM attached as an attestation; deployment verifies the signature.
- **DAST (optional, recommended):** OWASP ZAP baseline against the running stack in CI.
- **Updates:** Renovate/Dependabot (or an equivalent offline review) on a fixed cadence.

## 3. Frontend (TypeScript)
- **Location:** `/frontend` with its own `Dockerfile`, built as static assets served by unprivileged Nginx.
- **CSP-compatible code:** no inline `<script>`, no `eval`/`new Function`, no runtime CSS-in-JS injecting `<style>` without nonce. Prefer compiled CSS. Trusted Types enabled (`require-trusted-types-for 'script'`).
- **Zero-trust input:** typed Zod schema per form, shared with API types. Never `dangerouslySetInnerHTML`/`v-html` on untrusted data; DOMPurify if rich text is unavoidable.
- **Idempotency:** the API client attaches an `Idempotency-Key` (UUIDv4) to every POST/PUT/PATCH/DELETE, generated once per user action and reused on retries.
- **CSRF:** custom header (`X-Requested-With`) + CSRF token; backend also checks `Origin`/`Sec-Fetch-Site`. CORS allowlist only, never `*` with credentials.
- **Open redirects:** post-login `next`/`returnUrl` accepts only relative paths starting with a single `/`.
- **No secrets in the bundle;** no source maps in production; no sensitive data in `localStorage`/`sessionStorage`.

## 4. Backend (Python) & Database
- **Validation:** every body, query parameter, path parameter and header validated by Pydantic with max lengths and `extra="forbid"` (prevents mass assignment: `role`, `owner_id`, `is_admin` are never bindable from input).
- **Resource limits:** max request body size, max JSON depth/array length, max page size (e.g. 100), server-side timeouts. Regexes on user input are bounded or use a linear-time engine (`re2`) to avoid ReDoS.
- **Safe parsing:** no `pickle`, `marshal`, `yaml.load` (use `yaml.safe_load`), `eval`. XML via `defusedxml`. No user input in template sources (SSTI). File paths resolved and checked against a base directory (path traversal).
- **Idempotency store:** `(user_id, key) -> request_hash, status, response`, 24 h TTL. Same key + same hash returns the stored response; different hash returns `422`. This prevents duplicates; replay protection comes from TLS, CSRF tokens and short sessions.
- **SQL:** ORM or parameterized queries only; dynamic identifiers (`ORDER BY`, column names) through an allowlist.
- **Column-level encryption:** PII encrypted in the app with AES-256-GCM, envelope encryption (per-record DEK wrapped by a master key from the secrets provider), `key_version` column for rotation, HMAC-SHA256 blind index for searchable fields. Master key never stored in the DB or in DB backups.
- **Database hardening:** minimal PostgreSQL, `scram-sha-256`, TLS, two roles (migration owner, runtime DML-only), no superuser from the app, Alembic migrations, `statement_timeout` on the runtime role.
- **SQL console (if any):** read-only role, single `SELECT`, `statement_timeout`, row limit, locked configuration.
- **Cryptography & post-quantum:** AES-256-GCM at rest is quantum-resistant. PQ readiness targets key exchange and signatures: hybrid TLS `X25519MLKEM768` on the proxy; ML-KEM/ML-DSA (FIPS 203/204) only where the library supports them. Random values from `secrets`/`crypto.getRandomValues`. Never implement crypto primitives.
- **Outbound requests (SSRF):** any UI-configurable URL (webhooks, syslog, LLM, OIDC) is validated: `https` only (unless explicit allowlist), resolved IP rejected if loopback/link-local/metadata (`169.254.0.0/16`, `fd00:ec2::254`) or private unless allowlisted, re-checked after DNS resolution (DNS rebinding), no cross-host redirects, strict timeouts.
- **Outbound webhooks:** signed with HMAC-SHA256 (`X-Signature` + timestamp) so receivers can verify them.
- **Email:** header values stripped of CR/LF; SMTPS with certificate verification.
- **Exports:** CSV/XLSX cells starting with `=`, `+`, `-`, `@`, tab or CR are prefixed with `'` (formula injection).
- **Rate limiting:** login, password reset, MFA verification, API keys, exports, uploads and expensive endpoints.
- **Responses:** `Cache-Control: no-store` on authenticated API responses. Generic error messages with a `request_id`; stack traces only in logs. OpenAPI UI disabled or admin-only in production.

## 5. Authorization
- **Deny by default:** every route declares its required permission; a route without a declaration fails startup or tests.
- **RBAC** with roles mapped to permissions (not roles checked directly in code).
- **Object-level checks (IDOR/BOLA):** ownership or tenant check in the repository layer for every read/write by ID. Unauthorized access to an existing object returns `404`.
- **Function-level checks:** admin routes under `/api/v1/admin/*` guarded by a single dependency, never by hiding UI only.
- **Sensitive actions** (role changes, key creation, exports, deletion of users) require recent re-authentication; optional four-eyes approval for role escalation to admin.

## 6. File Uploads (if any)
- Size limit at proxy and app level; allowlist of extensions **and** magic-byte check.
- Stored outside the web root (or object storage), random file names, served with `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff`.
- Images re-encoded (strips EXIF and polyglots); archives checked for bombs and path traversal; optional offline antivirus (ClamAV).
- SVG and HTML uploads refused unless sanitized and served from an isolated origin.

## 7. Logging, Audit & Administration
- **Structured logs:** JSON, levels DEBUG/INFO/WARN/ERROR/CRITICAL, `request_id` everywhere, UTC timestamps, CR/LF stripped from user values. Passwords, tokens, keys, session IDs never logged.
- **Remote syslog:** RFC 5424 over TLS (RFC 5425) with bounded local buffer; host, port, TLS, CA configurable from the Admin UI. No rsyslog container.
- **Error notifications:** SMTPS and webhooks configurable from the Admin UI, SSRF-validated, "Send test" button, throttled.
- **PII masking:** middleware toggled from the Admin UI.
- **Tamper-evident audit trail:** append-only table (runtime role has `INSERT`/`SELECT` only), each row chained with the previous row's hash, forwarded to syslog. Covers sign-ins, failures, MFA changes, role changes, config changes, exports, API keys, honeytoken hits; records user, IP, user agent, UTC time and before/after values.
- **Admin dashboard:** usage statistics, errors, honeytoken alerts, blocked IPs.
- **Scheduled hygiene:** disable inactive accounts, purge expired tokens/sessions/idempotency keys, daily encrypted backups (key separate from the backup), retention policy and a tested restore procedure (`make restore-test`).
- **Personal data (GDPR):** retention periods per data type, user data export and deletion, purge jobs.

## 8. Authentication, Identity & Active Defense
- **Passwords:** Argon2id (argon2-cffi defaults / RFC 9106, minimum m=19 MiB, t=2, p=1), rehash on login when parameters change. Policy per NIST SP 800-63B: ≥ 12 characters, max ≥ 64, no composition rules, rejected if in a bundled breached/common passwords list.
- **Login:** identical response and timing for unknown users, progressive delays + per-account and per-IP throttling (avoid hard lockout that enables DoS).
- **Password reset:** single-use random token stored hashed, 30 min expiry, same response whether the account exists or not, all sessions revoked after reset.
- **Sessions:** server-side, cookie `__Host-session` (`HttpOnly; Secure; SameSite=Lax; Path=/`), ID rotated at login and privilege change, users can list and revoke their active sessions. No JWT in browser storage; if JWTs are used service-to-service: algorithm pinned, `none` refused, `aud`/`iss`/`exp` checked, short lifetime.
- **Admin sessions:** 1 h idle, 8 h absolute, re-authentication for destructive actions.
- **MFA:** TOTP (hashed recovery codes, replay of the same code refused) and WebAuthn/Passkeys. **MFA mandatory for admin accounts.**
- **Context detection:** admin sign-ins from unknown IP/device trigger a "New sign-in" notification and restrict destructive rights until MFA confirmation.
- **Bootstrap admin:** random password generated on first boot, readable once via `make admin-password`, forced reset + MFA enrollment at first login.
- **Break-glass account (optional):** local admin usable when SSO is down, disabled by default, every use raises a critical alert.
- **SSO:** OIDC code flow + PKCE, `state` and `nonce` validated, ID token signature/`aud`/`iss` checked; LDAPS or StartTLS with certificate verification, LDAP filter values escaped (LDAP injection). Directory groups mapped to roles.
- **SSO configuration, two equivalent sources:** every OIDC and LDAP(S) setting (enable flag, issuer/server URL, client ID, scopes, bind DN, base DN, user/group filters, CA bundle, group-to-role mapping, auto-provisioning) can be set **either** from the Admin UI **or** from environment variables (`BACKEND_OIDC_*`, `BACKEND_LDAP_*`).
  - Secrets (`client_secret`, bind password) come from `*_FILE` variables or the secrets provider, never plain env values in production; in the UI they are write-only (`••••last4`).
  - Precedence: a setting defined by environment wins and is shown **read-only** in the Admin UI with a "Managed by environment" badge; everything else is editable and stored encrypted in the database.
  - Same validation for both sources (Pydantic schema shared by env loader and admin API); invalid env configuration fails startup with a clear error, invalid UI input is rejected before saving.
  - The Admin UI has a "Test connection" action (OIDC discovery + JWKS fetch, LDAP bind + sample group search) and every change is audited.
  - Every variable is listed in `.env.example` (commented, disabled by default) and in the README configuration table, so a fully headless deployment never needs the UI. SSO stays optional: the app starts with local auth when nothing is configured.
- **API keys (inbound):** ≥ 32 random bytes, shown once, stored as SHA-256 hash with visible prefix, scoped, expiring, revocable, last-used date displayed.
- **Honeytokens (optional):** fake endpoints and fake credentials (e.g. a decoy admin account) that block the source IP and raise a critical alert. Client IP read only from trusted proxies (configured proxy count). Admins can unblock from the UI.

## 9. MCP Endpoints (only if requested)
Single source of truth for MCP; `offline-llm-orchestrator` refers to it.
- **User MCP (`/mcp`):** exposes application functions as tools, authenticated by user/service-account API key, going through the same RBAC, validation and rate limits as the REST API.
- **Admin MCP (`/mcp/admin`):** disabled by default, toggled from the Admin UI, admin-scoped key required, destructive tools require an explicit `confirm` argument.
- **Transport:** Streamable HTTP behind the proxy, `Origin` validated, rate limited per key.
- **Audit:** every tool call and resource read logged (key ID, tool, masked arguments, status, duration).

## 10. User Assistance
- Embedded help for users and administrators, sourced from `docs/help/`, bundled into the frontend.

## 11. README Security Content
The README follows the `project-blueprint` template. Its **Security** section contains:
- Applied security measures (short bullets).
- **Security & Compliance Scans** table: tool, version, image digest, result (e.g. `0 critical / 2 high`), date (`YYYY-MM-DD`). Never invent results: write `Not run yet` if not run.
- Deployment in ≤ 5 commands for Docker and rootless Podman.
- The Nginx configuration in a collapsed `<details>` block, identical to `deploy/nginx/nginx.conf` (based on [reference/nginx.conf](reference/nginx.conf)).

## 12. Pre-Production Cleanup
- `make clean-prod` (calls `scripts/clean-prod.sh`) after human validation: removes mock data and dummy seeds, runs `gitleaks` on tree and history, fails on any finding. Details in the `deep-cleanup` skill.

## Expected Output
1. Directory tree (per `project-blueprint`) and `docs/security/threat-model.md`.
2. `docker-compose.yml`, Dockerfiles, Nginx config.
3. Authentication flow diagrams (Mermaid sequence: local, OIDC, MFA, reset).
4. Logging/audit configuration.
5. `Makefile` with `scan`, `sbom`, `clean-prod`, `restore-test`.
6. Security tests from [reference/security-tests.md](reference/security-tests.md).
7. README following the blueprint template.

## Gotchas
- A `Dockerfile` without a numeric `USER` is a blocking defect.
- `docker compose up` must succeed on a clean machine with no `.env` and no hardcoded secret. Verify it from a fresh clone before declaring the scaffold done.
- Container-level seccomp profiles given by relative path are resolved differently by Docker Compose and podman-compose; rely on the runtime default profile plus the in-process filter.
- The audit gate must fail the build, not only print warnings.
- `SameSite=Strict` breaks the OIDC callback; use `Lax` + CSRF token.
- Encrypted columns cannot be searched or uniquely indexed without a blind index.
- Honeytokens without trusted-proxy configuration block your own reverse proxy.
- `verify=False`, `rejectUnauthorized: false`, `NODE_TLS_REJECT_UNAUTHORIZED=0` are forbidden; inject the internal CA instead.
- Hiding a button is not authorization: every check is server-side.
