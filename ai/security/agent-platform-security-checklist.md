# Agent Platform Security Checklist

**Source:** https://hi120ki.github.io/docs/ai-security/agent-platform-security-checklist/
**Saved:** 2026-06-03
**Tags:** ai, security, infrastructure, tools, checklist, compliance

---

## TL;DR
A systematic 8-section checklist for production AI agent platform builders covering sandbox isolation, network controls, credential management, observability, prompt filtering, memory security, supply chain, and platform access management.

## Key Concepts & Terms
- **Blast radius containment**: Running agent containers with read-only root filesystems, minimal write mounts, dropped Linux capabilities, and non-root users — so even `rm -rf /` only destroys the sandbox, not the host.
- **Credential injection proxy**: A proxy (e.g. WardGate) that intercepts outbound HTTP from the agent and attaches authentication headers before forwarding. The agent never sees real credentials — only the proxy URL.
- **Zombie Agents attack**: Multi-agent memory poisoning attack (arxiv 2602.15654) where a single poisoned memory entry injected via indirect prompt injection persists in shared LLM memory and propagates to all agents that read it.
- **Slopsquatting**: Supply chain attack exploiting LLM hallucinations — hallucinated package names get registered by attackers with malicious code, then installed when the agent runs `pip install`.
- **Confused Deputy Problem**: When a multi-user agent has credentials that exceed what any individual user is authorized to do — the agent becomes a confused deputy that can escalate privileges on behalf of attackers.

## Main Arguments & Takeaways
- **8 security domains to check**: (1) Sandbox isolation — one session per environment, Firecracker/Lambda/Cloud Run; (2) Network controls — egress allowlist by domain + HTTP-level proxy inspection for exfiltration; (3) Credential management — least privilege, short-lived credentials (1-hour expiry), injection proxy, signed commits without long-lived keys; (4) Observability — action logs, LLM API proxy logging (LiteLLM), agent framework tracing (Datadog/LangSmith/Arize), Falco for runtime security; (5) Prompt filtering — Model Armor or Bedrock Guardrails as one layer, not primary defense; (6) Memory security — namespace isolation per agent, audit logs for all reads/writes; (7) Supply chain — pin all dependencies to exact versions/content hashes, read-only config per session; (8) Platform access — identity-aware proxy, audit logs for all user interactions.
- **Prompt filtering is defense-in-depth, not primary defense**: Modern Agent Goal Hijack attacks use legitimate-sounding instructions with malicious intent that classifiers miss. Platform-level controls (sandbox, network, credentials) are the real mitigations.
- **Short-lived credentials are critical**: With LLM-assisted exploitation, attackers can escalate privileges within minutes of obtaining a credential. Long-lived PATs or API keys that leak through prompt injection remain exploitable for their entire lifetime.
- **Recoverability matters**: For every resource agents can modify or delete, a recovery mechanism must exist. Enforce Branch Rulesets to prevent direct pushes to default branch. For irreversible actions (sending emails), require Human in the Loop.

## Questions & Gaps
- How does the checklist weight these 8 domains against each other — are some foundational (sandbox + network) while others are defense-in-depth?
- The WardGate credential injection proxy is referenced — how mature is it? What's the operational burden?
- Zombie Agents attack requires shared memory across agents — is this a common architecture pattern or an edge case?
- No mention of cost/complexity tradeoffs for smaller teams — what's the minimum viable security posture?

## Related Notes
- [AI Agent Sandbox: How to Safely Run Autonomous Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/ai-agent-sandbox-safe-autonomous-agents-2026.md) — sandbox categories and best practices.
- [Code Sandboxes for LLMs: Docker vs gVisor vs Firecracker](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/code-sandboxes-llms-ai-agents-docker-gvisor-firecracker.md) — technical isolation comparison.
