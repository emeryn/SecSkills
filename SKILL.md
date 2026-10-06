---
name: secure-app-scaffolder
description: Scaffolds a secure-by-design Node.js/Python application compliant with ANSSI/CIS/FBI. Includes rootless execution, pre-build dependency audits, post-quantum DB encryption prep, advanced auth (OIDC/LDAPS/WebAuthn), strict logging, and air-gapped readiness.
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
- **Hardening:** Drop all unnecessary Linux capabilities (`cap_drop: - ALL`), mount root filesystems as read-only where possible, and enforce strict network segmentation.
- **Air-gapped Readiness:** The application MUST embed all necessary libraries and assets (no external CDNs or external library fetches at runtime) to ensure full functionality on isolated networks.
- **SBOM Generation:** Use minimalist Docker images (e.g., Alpine or Distroless). The CI/CD or build instructions must include generating an SBOM (Software Bill of Materials) report for each image after the build.
- **Deployment & Security README:** The `README.md` must ALWAYS specify the security measures applied and contain clear instructions for deploying the project via Docker, Podman, and kubectl, explicitly detailing the rootless configuration.
- **Secret Management (Optional Capability):** Scaffold the architecture to support fetching secrets dynamically from an external vault (e.g., HashiCorp Vault, SOPS) in production environments, keeping local `.env` usage strictly for basic local development.

### 2. Frontend (Node.js) & Reverse Proxies
- **Structure:** Scaffold the frontend code and its `Dockerfile` in a dedicated `/frontend` directory.
- **Supply Chain Security:** Include a pre-build step in the pipeline or `package.json` to verify dependencies are not compromised (e.g., `npm audit`, `yarn audit`, or `snyk test`) before any build command is executed.
- **Example Configurations (with Strict Headers):** ALWAYS create an `example/` folder containing secure, post-quantum resistant TLS configurations for Nginx (standalone/non-docker), Traefik (for Podman/Docker), and Nginx Ingress (for Kubernetes). These examples MUST include extreme security HTTP headers: a strict Content Security Policy (CSP) without `unsafe-inline`, HSTS with subdomains, and restrictive Permissions-Policy headers.
- **Session Security:** Enforce secure session management (Cookies must be `HttpOnly`, `SameSite=Lax`, `Secure`). Implement CSRF protection (e.g., requiring `X-Requested-With` headers) and ensure password changes invalidate all active sessions.

### 3. Backend (Python) & Database
- **Version & Environment:** Always specify the latest stable version of Python. Use virtual environments or precise container layers.
- **Security Scanning:** Include SAST and dependency checking tools in the dev requirements and pre-commit hooks.
- **Database (PostgreSQL Default):** Unless otherwise specified, ALWAYS default to a minimalist and hardened PostgreSQL base. If an SQL console is implemented, it MUST be sandboxed (read-only, single SELECT per query, locked configuration, time limit).
- **Post-Quantum Encryption at Rest:**
  - Scaffold backend libraries to handle **Encryption at Rest** natively at the application level.
  - Prepare for Post-Quantum (PQ) cryptography by recommending/implementing hybrid algorithms where supported by Python cryptography modules.

### 4. Logging, Audit & Operations
- **Centralized Logging:** Implement a robust logging system with explicit log levels (DEBUG, INFO, WARN, ERROR, CRITICAL).
- **Data Masking (Admin Toggle):** The logging configuration MUST include an optional Data Masking middleware (manageable via the admin interface) that automatically redacts PII (emails, phone numbers, etc.) from the logs to respect Privacy by Design (RGPD/CNIL).
- **Comprehensive Audit Trail:** Log all critical actions with IP, user, and detailed context (sign-ins, approvals, rights changes, data exports).
- **Syslog Connector (mTLS):** Include an `rsyslog` connector/configuration to securely forward logs via syslog (RFC 5424 structured data) over mutual TLS (mTLS).
- **PKI Injection:** Provide a mechanism (via the admin interface or environment/volume mapping) to inject an internal PKI PEM certificate for secure communications.
- **Scheduled Hygiene & Backups:** Scaffold automated tasks for account hygiene (disabling inactive accounts, expiring tokens) and daily encrypted database backups.

### 5. Authentication, Identity & Profiles
- **Password Hashing & Hardening:** The storage of local password hashes MUST ALWAYS use Argon2 (or stronger post-quantum). The login endpoint MUST mitigate timing attacks (identical response time for valid/invalid users) and brute force attacks (account lockout policies).
- **Local Auth Policy:** The default Admin account MUST have a forced password reset flag triggered on its very first login.
- **Profile Management:** The administrator MUST have the capability to define multiple user profiles/roles: User, Advanced User, and Administrator.
- **MFA / 2FA:** Scaffold support for OTP (One-Time Password / TOTP) and WebAuthn (FIDO2/Passkeys).
- **SSO Integration (OIDC & LDAPS):** 
  - OIDC MUST strictly use the authorization code flow with PKCE, state, nonce, and prevent open redirects.
  - LDAP(S) MUST operate strictly over TLS.
- **Service Accounts & API Tokens:** Support service accounts using revocable API tokens. Tokens must be displayed only once, stored as SHA3-256 fingerprints, and carry explicit expiration dates.
- **Auto-Provisioning Toggle:** The administrator MUST have the capability to enable or disable automatic account creation (auto-login) for new SSO/LDAP users. If disabled, new accounts must be placed in a "pending validation" state.
- **Group Management:** Both OIDC and LDAPS integrations must automatically map and associate directory groups to internal application roles.

### 6. User Assistance
- **Embedded Help:** The application MUST always embed integrated help or documentation sections for both users and administrators.

## Expected Output
Provide the directory tree (including the `example/` folder), the Dockerfiles (frontend/backend), the authentication workflow pseudocode or schemas (Python/Node), the logging configuration, and the security scanning commands.

## Gotchas
- Never output `Dockerfile`s without a `USER` directive. ANSSI/CIS strictly forbids root execution.
- Do not hardcode secrets or passwords; always use environment variables (`.env`) or secret managers.
- Ensure the pre-build frontend audit acts as a strict gate (exit code > 0 fails the build).