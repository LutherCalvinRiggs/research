# Introducing LangSmith Sandboxes: Secure Code Execution for Agents

**Source:** https://www.langchain.com/blog/introducing-langsmith-sandboxes-secure-code-execution-for-agents
**Saved:** 2026-06-03
**Tags:** ai, tools, sandboxing, infrastructure, llm, security

---

## TL;DR
LangSmith Sandboxes (Private Preview, March 2026) is LangChain's managed sandbox platform — microVM isolation, BYOD images, persistent state across sessions, sandbox pooling for warm starts, and Auth Proxy for secret isolation — tightly integrated with LangSmith's existing observability and deployment infrastructure.

## Key Concepts & Terms
- **Auth Proxy**: Sandboxes access external services through an Authentication Proxy, so secrets never touch the runtime. Credentials stay off the sandbox entirely — same pattern as sandcat and ExitBox's credential injection proxy.
- **Sandbox Templates**: Define image, CPU, and memory once; reuse for every invocation. Combine with BYOD (Bring Your Own Docker) images for fully custom environments.
- **Shared access**: Multiple agents can access the same sandbox, eliminating the need to transfer artifacts across isolated environments in multi-agent workflows.
- **Pooling and autoscaling**: Pre-provision warm sandboxes to eliminate cold start latency. Auto-scales as demand increases.
- **Tunnels**: Expose sandbox ports to the local machine for previewing agent output before deployment.

## Main Arguments & Takeaways
- **Traditional containers were designed for known, vetted code; agent-generated code is untrusted and unpredictable**: A web server handles a known set of operations. An agent might attempt anything, including malicious commands.
- **Building secure code execution yourself is expensive**: Spinning up containers, locking down network access, piping output back, tearing down, handling resource limits — then doing it at scale as more agents become coding agents.
- **Key workloads**: Coding assistant that validates its own output before responding; CI-style agent that clones a repo, installs dependencies, runs test suite before opening a PR (Open SWE); data analysis agent that executes Python against a dataset.
- **Tight LangSmith integration**: Same SDK as tracing and deployment. Native Deep Agents and Open SWE integrations. Sandbox calls traced alongside agent runs in the observability platform.
- **Coming next**: Shared volumes across sandboxes (Agent 1 writes, Agent 2 picks up); binary authorization (control which binaries can run, which domains are reachable); full execution tracing (every process and network call inside the VM as audit log).

## Questions & Gaps
- Private Preview — what's the waitlist timeline? What are the pricing/scale limits?
- How does LangSmith Sandboxes compare to E2B on isolation depth, cold start, and pricing at production scale?
- "Shared access" across multiple agents — how is this secured to prevent cross-agent data leakage?
- Binary authorization is listed as "coming next" — without it, how restrictive is the current execution environment?

## Related Notes
- [AI Agent Sandbox: How to Safely Run Autonomous Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/ai-agent-sandbox-safe-autonomous-agents-2026.md) — broader context; E2B and Modal alternatives compared.
- [Sandbox Container MCP Server](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/sandbox-container-mcp-server-secure-code-execution.md) — MCP-native approach to the same problem.
