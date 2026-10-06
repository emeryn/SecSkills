# 🛡️ Secure App Scaffolder - AI Skill

This repository contains a custom AI Skill (`SKILL.md`) designed to force AI assistants (like Claude) to generate **secure-by-design** application boilerplates. It enforces strict compliance with ANSSI, CIS, and FBI security guidelines.

By feeding this skill to your AI, it will stop generating generic, insecure code and instead output production-ready architectures featuring rootless execution, SBOM generation, post-quantum cryptography readiness, and strict identity management (OIDC/LDAPS).

## 🚀 How to use this skill with Claude

### Method 1: Using Claude Projects (Recommended)
If you have a Claude Pro or Team account, you can make this skill persistent for an entire workspace:
1. Open Claude and create a new **Project**.
2. Open the `SKILL.md` file from this repository and copy its entire content.
3. In your Claude Project, paste the content into the **Custom Instructions** field.
4. Start a new chat within that project. Claude will now strictly apply these rules to any architecture you ask it to generate.

### Method 2: Direct Chat (Standard Prompting)
If you don't use Claude Projects, you can inject the skill manually at the start of your conversation:
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
- **Post-Quantum Cryptography:** Prepares PostgreSQL and Nginx for hybrid PQ encryption algorithms.
- **Enterprise Auth:** Enforces Argon2 (or stronger), WebAuthn, OIDC (PKCE), and LDAPS.
- **Strict Audit & Logging:** Generates mTLS Syslog configurations and comprehensive audit trails.