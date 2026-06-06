# ExitBox: Run AI Coding Agents in Complete Isolation

**Source:** https://medium.com/@cloud-exit/introducing-exitbox-run-ai-coding-agents-in-complete-isolation-6013fb5bdd06
**Saved:** 2026-06-03
**Tags:** ai, security, tools, sandboxing, oss, infrastructure

---

## TL;DR
ExitBox (Cloud Exit) is an open-source CLI tool that runs AI coding agents (Claude Code, Codex, OpenCode) in rootless containers with layered network isolation — DNS cut off inside the container, all traffic through a Squid proxy enforcing a domain allowlist, runtime domain approval workflow, and an encrypted AES-256 credential vault with per-access host approval.

## Key Concepts & Terms
- **Defense in depth (4 layers)**: (1) DNS disabled inside container — agent can't resolve external names; (2) Squid proxy is the only egress path — tools must use it or get nothing; (3) Proxy enforces domain allowlist — everything else blocked; (4) Linux capabilities dropped, privilege escalation blocked, raw sockets disabled.
- **exitbox-allow**: A command the agent itself can run when it hits a blocked domain. Sends an approval request over Unix socket IPC to the host terminal. Operator approves/denies; approved domains hot-reload into Squid without container restart.
- **Encrypted vault**: AES-256 with Argon2id key derivation (OWASP parameters). Each workspace gets its own vault. Every read/write triggers a y/n approval popup via tmux on the host. `.env` files are mounted as `/dev/null` inside the container to force vault usage.
- **Named resumable sessions**: Sessions have names and persist state. `exitbox run claude --name "my-feature"` resumes an existing session automatically.
- **BadgerDB**: Embedded Go key-value store for session metadata — no separate database process, compiles into the single binary, handles write-ahead log recovery for unclean container shutdowns.

## Main Arguments & Takeaways
- **The professional vs. hobby distinction**: Solo side-project → acceptable to run agents on host. Professional setting with customer data, production credentials, or proprietary code on the same machine → not acceptable. ExitBox was built to enforce this policy company-wide at Cloud Exit.
- **Agents inherit dangerous shell context by default**: Shell environment variables contain API keys, database connection strings, cloud provider credentials. Agents have read/write access to the entire home directory. No outbound request restrictions by default.
- **Autonomy amplifies the risk**: Features like automatic tool use, multi-step reasoning, and background execution mean agents act without waiting for approval — every action runs with whatever permissions they inherited.
- **Runtime domain approvals solve the strict-allowlist UX problem**: A completely static allowlist breaks workflows constantly. `exitbox-allow` lets the agent request domains at runtime with a single host-side approval, keeping the allowlist strict without making it painful.
- **Supply chain hardening**: Claude Code installed via direct binary download with SHA-256 checksum verification against Anthropic's signed manifest. Alpine Linux base (~5MB vs ~80MB Debian slim). Three-layer image build (base/agent/project) with label-based caching.
- **AGPL-3.0 with commercial licensing available**: Open source, Podman-first (rootless, daemonless), Docker compatible.


## Code Examples

### Basic Usage
```bash
# Install
cargo install --git https://github.com/cloud-exit/exitbox

# Start an isolated Claude Code session
exitbox run claude --name my-feature   --allow api.anthropic.com   --allow github.com   --allow api.github.com

# Resume a named session (picks up where you left off)
exitbox run claude --name my-feature

# Start OpenCode instead
exitbox run opencode --name backend-work --allow api.openai.com
```

### The Four Isolation Layers in Practice
```bash
# Layer 1 — DNS disabled inside container
# Agent tries: curl https://malicious.com → fails at DNS resolution
# No /etc/resolv.conf inside the container — all DNS queries fail

# Layer 2 — Squid proxy is the only egress path
# HTTPS_PROXY=http://squid:3128 is set for all processes
# Tools that don't use the proxy get no network access

# Layer 3 — Domain allowlist enforced at proxy
# Squid config (auto-generated from --allow flags):
#   acl allowed_domains dstdomain api.anthropic.com github.com api.github.com
#   http_access allow allowed_domains
#   http_access deny all

# Layer 4 — Linux capabilities dropped
# docker run --cap-drop=ALL --security-opt=no-new-privileges
# --security-opt=seccomp=exitbox-seccomp.json (raw sockets blocked)
```

### Runtime Domain Approval (exitbox-allow)
```bash
# Inside the container, agent hits a blocked domain and calls:
exitbox-allow pypi.org
# This sends a request over Unix socket IPC to the host terminal

# On the HOST, you see:
# ┌─────────────────────────────────────┐
# │ Agent requests access to: pypi.org  │
# │ Allow? [y/n]                        │
# └─────────────────────────────────────┘
# Press y → domain hot-reloaded into Squid, no container restart
# Press n → agent gets 403, continues without it
```

### Credential Vault
```bash
# Store a secret in the workspace vault (AES-256, Argon2id)
exitbox vault set STRIPE_SECRET_KEY --workspace my-feature
# Prompts for value, encrypts, stores in ~/.exitbox/vaults/my-feature/

# Inside the container, agent reads from vault (not .env):
# Every read triggers a host-side approval popup:
# ┌─────────────────────────────────────────────────────┐
# │ Agent is reading: STRIPE_SECRET_KEY                 │
# │ Will be sent to: api.stripe.com (in allowlist)      │
# │ Allow? [y/n]                                        │
# └─────────────────────────────────────────────────────┘

# .env is explicitly blocked:
# docker run -v /dev/null:/workspace/.env:ro ...
```
## Questions & Gaps
- How does ExitBox handle agent tools that bundle their own Node.js or CA roots (like Cursor CLI) — the same TLS interception problem sandcat documents?
- The 4-layer defense uses Squid for egress — how does this perform under concurrent agent load?
- Named sessions with BadgerDB — what's the recovery story for a corrupted vault vs. corrupted session state?
- AGPL-3.0 licensing — what are the practical implications for enterprises using it internally vs. embedding in products?

## Related Notes
- [Sandcat: Docker Dev Container for AI Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/sandcat-docker-devcontainer-ai-agent-sandbox.md) — similar credential proxy + network isolation approach via devcontainer + mitmproxy rather than a standalone CLI.
- [AI Agent Sandbox: How to Safely Run Autonomous Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/ai-agent-sandbox-safe-autonomous-agents-2026.md) — broader sandbox taxonomy and best practices.
