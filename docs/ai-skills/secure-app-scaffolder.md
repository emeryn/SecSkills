---
name: secure-app-scaffolder
description: Scaffolds a secure-by-design Node.js/Python application compliant with ANSSI/CIS/FBI. Includes rootless execution, advanced auth, strict logging, admin dashboards, defense in depth (seccomp, column-level encryption, idempotency), optional MCP endpoints, and post-validation cleanup.
---
# Secure App Scaffolder

This skill generates a complete, secure-by-design application boilerplate compliant with ANSSI, CIS, and FBI recommendations. It scaffolds a Node.js frontend, a Python backend, and prepares a PostgreSQL database configuration with advanced security measures.

## When to Use
- The user asks to start, scaffold, initialize, or architect a new secure web application.
- The user requests a boilerplate with ANSSI, CIS, or FBI compliance.
- The user wants a rootless Node/Python/Postgres stack with advanced auth (OIDC, LDAPS, WebAuthn) and strict security/logging patterns.

## Steps

When triggered, generate the project structure and essential configuration files following these strict rules:

### 1. Global Architecture & Execution
- **Rootless Execution:** All generated `Dockerfile` and `docker-compose.yml` (or Podman manifests) MUST run as non-root users (`USER 1000:1000` or equivalent unprivileged user).
- **Out-of-the-box Defaults:** The `docker-compose.yml` MUST function immediately using safe default values without requiring manual `.env` file editing just to start. The setup and start process MUST NOT exceed 5 terminal commands.
- **Hardening & Syscalls:** Drop all unnecessary Linux capabilities (`cap_drop: - ALL`), mount root filesystems as read-only where possible, enforce strict network segmentation, AND impose execution hardening (e.g., `security_opt: - no-new-privileges` and Seccomp profiles) to strictly forbid the backend from launching shell sub-processes (`execve`).
- **Air-gapped Readiness:** The application MUST embed all necessary libraries and assets (no external CDNs or external library fetches at runtime) to ensure full functionality on isolated networks.
- **Dual Security Scanning & SBOM:** The CI/CD or build instructions MUST include a dual-scan approach:
  1. A container-native scanner (e.g., Trivy or Grype) to generate the SBOM (Software Bill of Materials) and scan for CVEs in app dependencies.
  2. `oscap-podman` (or `oscap-docker`) to evaluate the built image against OS-level CIS configuration baselines.
- **Secret Management (Optional Capability):** Scaffold the architecture to support fetching secrets dynamically from an external vault (e.g., HashiCorp Vault, SOPS) in production environments, keeping local `.env` usage strictly for basic local development.

### 2. Documentation & README
- **Deployment & Security README:** The generated `README.md` MUST be strictly technical and concise. Do NOT add unnecessary fluff, marketing text, or unsolicited ideas/features. Do NOT include details about the host machine executing the generation. It MUST ALWAYS include:
  - A text-based functional architecture diagram (ASCII or Mermaid).
  - A summary of backend features and a clear list of available API paths.
  - A specific **"Security & Compliance Scans"** section explicitly declaring the results of the Trivy and OpenSCAP scans, including the exact date of the last check.
  - The security measures applied.
  - Clear instructions for deploying the project in **max 5 commands** via Docker, Podman, and kubectl, detailing the rootless configuration.
  - A **Secure Nginx Configuration Snippet:** Provide a highly secure example of an Nginx configuration block directly in the README (incorporating strict CSP without `unsafe-inline`, HSTS with subdomains, post-quantum readiness, and restrictive Permissions-Policy headers).

### 3. Frontend (Node.js) & Reverse Proxies
- **Structure:** Scaffold the frontend code and its `Dockerfile` in a dedicated `/frontend` directory.
- **Zero-Trust Input (XSS & Injection Prevention):** Every single input field, textbox, and form MUST be strictly typed and sanitized. Enforce context-aware output encoding to prevent XSS and implement strict client-side validation schemas.
- **Idempotency (Anti-Replay):** The frontend MUST generate and include an `Idempotency-Key` header for all state-changing API requests (POST/PUT/DELETE) to prevent replay attacks and accidental duplicates.
- **Supply Chain Security:** Include a pre-build step in the pipeline or `package.json` to verify dependencies are not compromised (e.g., `npm audit`, `yarn audit`, or `snyk test`) before any build command is executed.
- **Session Security:** Enforce secure session management (Cookies must be `HttpOnly`, `SameSite=Lax`, `Secure`). Implement CSRF protection (e.g., requiring `X-Requested-With` headers) and ensure password changes invalidate all active sessions.

### 4. Backend (Python) & Database
- **Version & Environment:** Always specify the latest stable version of Python. Use virtual environments or precise container layers.
- **Zero-Trust Validation & Idempotency:** The backend MUST implement strict input validation (e.g., Pydantic, Zod, or equivalent) for ALL incoming payloads. It MUST cache and validate the `Idempotency-Key` to ignore duplicate state-changing requests. All database interactions MUST strictly use parameterized queries or a safe ORM to categorically eliminate SQL Injection.
- **Column-Level Encryption (Data in Transit Internally):** The backend MUST encrypt highly sensitive fields (PII, emails, phones) at the application level BEFORE sending them to PostgreSQL. The DB must only store ciphertext for these fields, rendering raw DB dumps useless without the application's master key.
- **Security Scanning:** Include SAST and dependency checking tools in the dev requirements and pre-commit hooks.
- **Database (PostgreSQL Default):** Unless otherwise specified, ALWAYS default to a minimalist and hardened PostgreSQL base. If an SQL console is implemented, it MUST be sandboxed (read-only, single SELECT per query, locked configuration, time limit).
- **Post-Quantum Encryption at Rest:** Prepare for Post-Quantum (PQ) cryptography by recommending/implementing hybrid algorithms where supported by Python cryptography modules.

### 5. Logging, Audit & Administration
- **Centralized Logging:** Implement a robust logging system with explicit log levels (DEBUG, INFO, WARN, ERROR, CRITICAL).
- **Remote Syslog Forwarding (Admin Configurable):** The application MUST act as a syslog client capable of forwarding logs to a remote server. Do NOT deploy a standalone rsyslog container. The destination server details (host, port, TLS) MUST be fully configurable directly from the Admin UI.
- **Error Notifications:** Implement an alerting system for application errors, dispatching notifications via SMTPS and Webhooks. Both the SMTP server settings and Webhook endpoints MUST be fully configurable from the Admin UI.
- **Data Masking (Admin Toggle):** The logging configuration MUST include an optional Data Masking middleware (manageable via the admin interface) that automatically redacts PII.
- **Comprehensive Audit Trail:** Log all critical actions with IP, user, and detailed context (sign-ins, approvals, rights changes, data exports).
- **Usage & Defense Dashboard:** The Admin interface MUST include a dashboard capable of extracting and visualizing application usage statistics, as well as visualizing alerts triggered by the Active Defense Honeytokens.
- **Scheduled Hygiene & Backups:** Scaffold automated tasks for account hygiene (disabling inactive accounts, expiring tokens) and daily encrypted database backups.

### 6. Authentication, Identity, Profiles & Active Defense
- **Password Hashing & Hardening:** The storage of local password hashes MUST ALWAYS use Argon2 (or stronger post-quantum). The login endpoint MUST mitigate timing attacks and brute force attacks (account lockout policies).
- **Admin Session Idle Timeout:** Admin user sessions MUST enforce a strict automatic expiration after exactly 1 hour of inactivity.
- **Behavior Detection & Fingerprinting:** The system MUST analyze the context of admin sessions (IP, device fingerprint). Logins from unrecognized contexts with valid credentials must trigger a "New Sign-in" notification and temporarily restrict destructive rights.
- **Active Defense / Honeytokens (Optional API):** Optionally scaffold fake API endpoints (e.g., `/api/v1/legacy-admin`). Any access to these traps by scanners/attackers must instantly block the attacker's IP and trigger a critical alert visible in the Admin UI.
- **Local Auth Policy:** The default Admin account MUST have a forced password reset flag triggered on its very first login.
- **MFA / 2FA:** Scaffold support for OTP (One-Time Password / TOTP) and WebAuthn (FIDO2/Passkeys).
- **SSO Integration (OIDC & LDAPS):** OIDC MUST strictly use the authorization code flow with PKCE. LDAP(S) MUST operate strictly over TLS.
- **Group Management:** Both OIDC and LDAPS integrations must automatically map and associate directory groups to internal application roles.

### 7. Optional MCP (Model Context Protocol) Integration
- **Dedicated Endpoints:** If MCP integration is requested, the application MUST provide two distinct and dedicated endpoints:
  1. **Admin MCP Endpoint:** A highly privileged endpoint strictly reserved for Administrators. It must be toggleable (enable/disable) via the Admin UI and allows end-to-end piloting and configuration of the application.
  2. **User MCP Endpoint:** A standard endpoint allowing authenticated users (or service accounts) to interact with the application. Its capabilities MUST be strictly restricted by the user's standard RBAC permissions.

### 8. User Assistance
- **Embedded Help:** The application MUST always embed integrated help or documentation sections for both users and administrators.

### 9. Post-Validation Cleanup (Pre-Production)
- **Sanitization Process:** The scaffolding MUST include a script or a documented procedure (e.g., `make clean-prod` or `cleanup.sh`) to be executed after human validation of the project.
- **Data Wiping:** This process MUST systematically delete all example data, mock files, and dummy database seeds used during the scaffolding or testing phase.
- **Secret & PII Scanning:** The cleanup process MUST include a final automated scan (using tools like `trufflehog`, `gitleaks`, or custom regex) to verify that absolutely no hardcoded secrets, API keys, or Personally Identifiable Information (PII) remain in the codebase before it is deployed to production.

## Expected Output
Provide the directory tree, the Dockerfiles, the authentication workflow schemas, the logging configuration, and the security scanning commands.

## Gotchas
- Never output `Dockerfile`s without a `USER` directive. ANSSI/CIS strictly forbids root execution.
- Ensure the pre-build frontend audit acts as a strict gate (exit code > 0 fails the build).
- The `docker-compose.yml` MUST run successfully right away—no initial `.env` setup overhead.