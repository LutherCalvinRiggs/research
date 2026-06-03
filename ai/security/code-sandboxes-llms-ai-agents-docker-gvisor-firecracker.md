# Code Sandboxes for LLMs and AI Agents: Docker, gVisor, Firecracker

**Source:** https://www.codeant.ai/blogs/agentic-rag-shell-sandboxing
**Saved:** 2026-06-03
**Tags:** ai, security, sandboxing, infrastructure, prompting, llm

---

## TL;DR
Application-layer defenses (prompt injection classifiers, input sanitization, output filtering) all have exploitable bypasses when AI agents have shell access. System-level isolation via Firecracker microVMs is the only architecturally sound defense.

## Key Concepts & Terms
- **Agentic RAG**: Retrieval-Augmented Generation where the LLM actively calls shell commands to explore a codebase, rather than passively receiving pre-chunked embeddings. Powerful but introduces a direct attack surface.
- **Prompt injection classifier**: An application-layer defense that flags malicious text before passing it to the agent. Fails on obfuscated payloads (base64-encoded commands, indirect instructions).
- **Input sanitization**: Blocking dangerous shell operators or binaries. Bypassed when an allowed tool (like pytest) can execute arbitrary code via conftest.py files.
- **Output sanitization**: Masking leaked secrets in agent output. Bypassed when attackers instruct the agent to encode secrets in a format that evades pattern matchers.

## Main Arguments & Takeaways
- **All application-layer defenses are bypassable** — they can slow attackers but none provide true isolation. If the model can run commands, it can escape through creative execution paths.
- **The base64 attack**: Attacker embeds `echo "base64payload" | base64 -d | sh` in a prompt disguised as a legitimate task. Classifiers miss it because the malicious command is obfuscated.
- **The pytest attack**: Attacker plants malicious code in conftest.py before asking the agent to "run unit tests." Input sanitization allows pytest but the tests execute arbitrary Python.
- **The output encoding attack**: Attacker asks the agent to base64-encode secrets "for safety" — the short encoded string bypasses pattern matchers designed for raw tokens.
- **Firecracker wins for AI agent isolation**: Full kernel isolation, fresh VM per session, deterministic cleanup, built-in network separation, proven at hyperscale (AWS Lambda). The only option for zero-trust agent environments.
- **Comparison summary**: Docker (fast, shared kernel, insufficient for untrusted code) → gVisor (filtered syscalls, ~70 vs 300+ Linux syscalls, moderate protection) → Firecracker (separate kernel per execution, maximum isolation, ~125ms boot).

## Notable Quotes
> "LLMs don't inherently understand boundaries. A task disguised as harmless can result in credential leakage."

## Questions & Gaps
- gVisor's 10-30% CPU overhead — at what scale does this become economically significant vs. Firecracker?
- The article focuses on shell access as the threat surface — how does the threat model change for browser-using agents vs. code-executing agents?
- Are there formal evaluations of how many real-world prompt injection attacks are caught by classifiers vs. how many bypass them?
