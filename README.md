# 🛡️ Secure App Scaffolder - AI Skill

This repository contains a custom AI Skill (`SKILL.md`) designed to force AI assistants (like Claude) to generate **secure-by-design** application boilerplates. It enforces strict compliance with ANSSI, CIS, FBI, and GDPR/CNIL security guidelines.

By feeding this skill to your AI, it will stop generating generic, insecure code and instead output production-ready architectures featuring rootless execution, SBOM generation, post-quantum cryptography readiness, and strict identity management.

## 🚀 How to use this skill with Claude

### Method 1: VS Code Integration (Claude Code, Cline, Roo)
If you are using Claude directly inside your IDE, you can make this skill run automatically:
- **Official Claude Code:** Rename the file to `.clauderules` and place it at the root of your project.
- **Cline (formerly Claude Dev):** Rename the file to `.clinerules` at the root of your project.
- **Roo Code:** Rename the file to `.roomodes` at the root of your project.
- **Continue.dev (Chat):** Place the file in `docs/secure-app-scaffolder.md` and reference it in your chat using `@docs/secure-app-scaffolder.md`.

### Method 2: Using Claude Projects (Web Interface)
If you have a Claude Pro or Team account, you can make this skill persistent for an entire workspace:
1. Open Claude and create a new **Project**.
2. Open the `SKILL.md` file from this repository and copy its entire content.
3. In your Claude Project, paste the content into the **Custom Instructions** field.
4. Start a new chat within that project. Claude will now strictly apply these rules.

### Method 3: Direct Chat (Standard Prompting)
If you don't use Claude Projects or an IDE extension, inject the skill manually:
1. Copy the entire content of `SKILL.md`.
2. Open a new chat in Claude.
3. Paste the content and append your actual request at the end. 

**Example prompt:**
> [Paste SKILL.md content here]
> 
> ***
> Using the Secure App Scaffolder skill above, generate the architecture and Dockerfiles for a new HR Intranet web application.

## ✨ Features enforced by this Skill
- **Zero Root Execution:** Forces unprivileged users (`USER 1000:1000`) in all container engines.
- **Air-Gapped Readiness:** No external CDNs at runtime.
- **Supply Chain Security:** Blocks compilation if vulnerabilities are found in Node dependencies and enforces SBOM generation.
- **Dynamic Secret Management:** Scaffolds Vault/SOPS integration to eliminate `.env` files in production.
- **Post-Quantum Cryptography:** Prepares PostgreSQL and Nginx for hybrid PQ encryption algorithms.
- **Enterprise Auth:** Enforces Argon2 (or stronger), WebAuthn, OIDC (PKCE), and LDAPS.
- **Extreme HTTP Security:** Enforces strict CSP (no `unsafe-inline`), HSTS, and Permissions-Policy in reverse proxies.
- **Privacy by Design & Audit:** Generates mTLS Syslog configurations, comprehensive audit trails, and PII Data Masking middlewares.