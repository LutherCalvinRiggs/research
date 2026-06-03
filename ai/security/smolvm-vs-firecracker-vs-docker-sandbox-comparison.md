# SmolVM vs Firecracker vs Docker: Sandboxing AI-Generated Code

**Source:** https://particula.tech/blog/smolvm-vs-firecracker-sandbox-ai-generated-code
**Saved:** 2026-06-03
**Tags:** ai, security, sandboxing, infrastructure, microvm, tools

---

## TL;DR
SmolVM (launched April 17, 2026) is the first microVM that works on macOS developer machines with sub-200ms cold starts, closing the gap between Firecracker's security and Docker's developer ergonomics — but it's new and not yet production-hardened.

## Key Concepts & Terms
- **SmolVM**: A single-executable microVM runtime using Hypervisor.framework (macOS) and libkrun (Linux). Sub-200ms cold starts, designed for ephemeral per-agent-invocation sandboxing. Launched April 17, 2026.
- **Hypervisor.framework**: macOS native hypervisor API (same one Docker Desktop uses). Lets SmolVM run real hardware-virtualized VMs natively on Mac without a separate Linux host.
- **libkrun**: Red Hat's project for running OCI images in a microVM on Linux. SmolVM's Linux backend.
- **Copy-on-write memory isolation**: How Freestyle implements memory forking — not a full snapshot, but deferred copying so fork cost is O(1) with respect to both VM size and number of forks.
- **Slopsquatting**: Supply chain attack where hallucinated package names (packages the LLM invented but don't exist) are registered by attackers with malicious code.

## Main Arguments & Takeaways
- **Docker's shared kernel is consistently exploitable**: CVE-2024-21626 (runc), CVE-2025-23359 (NVIDIA Container Toolkit), CVE-2026-1109 (kernel io_uring) each broke guest/host isolation in 2024-2026. One successful exploit = root on the host.
- **Cold start latency drives teams to skip sandboxing entirely**: Docker's 500ms-2s cold start on the hot path of a real-time coding agent causes teams to quietly remove containerization. SmolVM's 80-260ms makes sandboxing practical on every request.
- **Decision framework**: SmolVM → per-request ephemeral AI code execution, any OS. Firecracker → heavy tenant workloads on Linux, 100+ VMs per host, need snapshot prewarming. Docker → cached build steps, trusted code. gVisor → untrusted HTTP workloads where syscall overhead is acceptable.
- **Production gotchas**: Network egress allowlists (allowing pypi.org enables DNS exfiltration via package names — pre-install dependencies instead); writable scratch mounts create cross-invocation side channels (use ephemeral tmpfs); GPU passthrough doesn't work well in cross-platform microVMs; cold start regresses as image size grows.
- **The recipe that works in production**: Hardware-virtualized microVM per invocation + deny-all network with explicit allowlist + read-only code mount + hard budget (CPU, memory, wall-clock) + audit every exec.

## Notable Quotes
> "Three days of running a Claude Code instance through a microVM sandbox is worth more than a quarter of security-review meetings about 'what if the agent writes something bad.'"

## Questions & Gaps
- SmolVM has no escape-class CVEs yet but is brand new — how will its security posture hold up as it gains adoption and scrutiny?
- The "80-260ms" cold start range is wide — what drives the variance?
- GPU workload sandboxing remains unsolved for cross-platform microVMs — is there a viable path forward?
- How does SmolVM perform under concurrent load (e.g. 50 simultaneous agent invocations on a laptop)?

## Related Notes
- [Self-Hosted AI Sandboxes Guide](https://github.com/LutherCalvinRiggs/research/blob/main/ai/infrastructure/self-hosted-ai-sandboxes-guide-2026.md) — decision framework for deployment approaches.
- [Meta/AWS Graviton5/Firecracker](https://github.com/LutherCalvinRiggs/research/blob/main/ai/infrastructure/meta-aws-graviton5-firecracker-agentic-ai.md) — Firecracker at hyperscale in production.
