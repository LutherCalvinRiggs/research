# Sandcat: Docker Dev Container Setup for Securely Running AI Agents

**Source:** https://github.com/VirtusLab/sandcat
**Saved:** 2026-06-03
**Tags:** ai, security, tools, infrastructure, sandboxing, oss

---

## TL;DR
Sandcat (VirtusLab) is an open-source Docker + devcontainer setup that routes all AI agent traffic through a transparent mitmproxy via WireGuard, enforcing network allowlists and injecting secrets at the proxy level so the agent container never sees real credentials.

## Key Concepts & Terms
- **Transparent mitmproxy**: All container traffic is intercepted by mitmproxy (running in WireGuard mode) without any per-tool proxy configuration inside the container. HTTP/S, DNS, and all TCP/UDP traffic is captured.
- **Secret substitution**: Real API keys never reach the container. Environment variables contain deterministic placeholders (`SANDCAT_PLACEHOLDER_ANTHROPIC_API_KEY`). The mitmproxy addon replaces placeholders with real values only when requests go to allowed hosts.
- **Leak detection**: If a placeholder appears in a request to a host NOT in the allowlist, mitmproxy blocks the request with HTTP 403. Prevents accidental secret leakage to unintended services.
- **WireGuard kill switch**: iptables rules ensure all traffic must go through the WireGuard tunnel to mitmproxy. Direct eth0 access is blocked. If the tunnel drops, the container loses network access entirely — no bypass possible.

## Main Arguments & Takeaways
- **Architecture**: Three containers — `mitmproxy` (WireGuard server + network rules + secret substitution), `wg-client` (WireGuard client + iptables kill switch, the only container with NET_ADMIN), `agent` (shares wg-client's network namespace, no NET_ADMIN).
- **Secrets never leave mitmproxy**: The agent sees only placeholders. Real values are stored in mitmproxy's settings file (host-side, read-only bind-mount). On each outbound request, mitmproxy checks both the network allowlist and the per-secret host allowlist before substituting.
- **VS Code devcontainer hardening**: Credential socket cleanup (SSH_AUTH_SOCK, GPG_AGENT_INFO cleared), workspace trust enabled, local terminal creation blocked, .devcontainer directory mounted read-only (agent can't modify its own sandbox config).
- **1Password integration**: Secrets can reference `op://` vault paths instead of plaintext values. The mitmproxy-op image includes the `op` CLI and resolves references at startup via a service account token.
- **Claude Code in dangerous mode**: The devcontainer.json enables `claudeCode.allowDangerouslySkipPermissions` — because sandcat provides the real security boundary (network isolation, secret substitution, kill switch), in-container permission prompts add friction without meaningful security benefit.
- **Default allowlist is permissive**: By default, all GET traffic is allowed (prompt injection vector) and full GitHub access is granted (exfiltration vector). The README acknowledges this explicitly and recommends tightening for production.


## Code Examples

### Three-Container Architecture
```yaml
# docker-compose.yml (simplified from VirtusLab/sandcat)
services:
  mitmproxy:
    image: mitmproxy/mitmproxy
    # WireGuard server + secret substitution addon
    volumes:
      - ./secrets.yaml:/secrets.yaml:ro      # real API keys, read-only
      - ./allowlist.yaml:/allowlist.yaml:ro  # domain allowlist
    cap_add: [NET_ADMIN]
    environment:
      - MITM_ADDON=/addons/secret_substitution.py

  wg-client:
    image: sandcat/wg-client
    # WireGuard client + iptables kill switch
    cap_add: [NET_ADMIN]
    network_mode: host
    # iptables: all traffic must go through WireGuard tunnel
    # direct eth0 access is blocked — if tunnel drops, container loses network

  agent:
    image: ghcr.io/anthropics/claude-code:latest
    network_mode: "service:wg-client"  # shares wg-client network namespace
    # No NET_ADMIN — cannot modify its own network rules
    volumes:
      - ./workspace:/workspace:rw
      - ./deploy:/workspace/deploy:ro        # read-only sensitive dirs
      - /dev/null:/workspace/.env:ro         # .env blocked entirely
    environment:
      # Placeholder — mitmproxy substitutes real value only for allowed hosts
      - ANTHROPIC_API_KEY=SANDCAT_PLACEHOLDER_ANTHROPIC_API_KEY
```

### Secret Substitution Pattern (mitmproxy addon)
```python
# addons/secret_substitution.py
# Runs inside mitmproxy — agent never sees this code
import yaml
from mitmproxy import http

with open("/secrets.yaml") as f:
    secrets = yaml.safe_load(f)   # { ANTHROPIC_API_KEY: "sk-ant-..." }

with open("/allowlist.yaml") as f:
    allowlist = yaml.safe_load(f) # { ANTHROPIC_API_KEY: ["api.anthropic.com"] }

def request(flow: http.HTTPFlow):
    for placeholder, real_value in secrets.items():
        if placeholder in str(flow.request.content):
            allowed_hosts = allowlist.get(placeholder, [])
            if flow.request.pretty_host not in allowed_hosts:
                # Placeholder appeared in request to non-allowed host — block it
                flow.response = http.Response.make(403,
                    f"Secret leak blocked: {placeholder} → {flow.request.pretty_host}")
                return
            # Allowed host — substitute real value
            flow.request.content = flow.request.content.replace(
                placeholder.encode(), real_value.encode())
```

### devcontainer.json Hardening
```json
{
  "name": "sandcat-agent",
  "image": "ghcr.io/anthropics/claude-code:latest",
  "runArgs": [
    "--cap-drop=ALL",
    "--no-new-privileges",
    "--security-opt=no-new-privileges:true"
  ],
  "containerEnv": {
    "ANTHROPIC_API_KEY": "SANDCAT_PLACEHOLDER_ANTHROPIC_API_KEY",
    "SSH_AUTH_SOCK": "",
    "GPG_AGENT_INFO": ""
  },
  "mounts": [
    "source=${localWorkspaceFolder}/deploy,target=/workspace/deploy,type=bind,readonly",
    "source=/dev/null,target=/workspace/.env,type=bind,readonly"
  ],
  "settings": {
    "claudeCode.allowDangerouslySkipPermissions": true
  }
}
```
## Questions & Gaps
- The default settings are intentionally permissive — what's the recommended locked-down configuration for handling customer data?
- VS Code's devcontainer integration has known escape vectors (credential socket discovery) that sandcat mitigates best-effort — how complete is this mitigation?
- How does mitmproxy handle mTLS or certificate pinning in agent tools that bundle their own CA roots (e.g. Cursor CLI)?
- No mention of auditability — can you replay exactly what the agent did after the fact?

## Related Notes
- [ExitBox: Run AI Coding Agents in Complete Isolation](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/exitbox-ai-coding-agents-complete-isolation.md) — similar approach (network isolation + secret proxy) but as a standalone CLI tool rather than devcontainer setup.
- [Agent Platform Security Checklist](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/agent-platform-security-checklist.md) — sandcat addresses several checklist items: credential injection proxy, network allowlisting, read-only config.
