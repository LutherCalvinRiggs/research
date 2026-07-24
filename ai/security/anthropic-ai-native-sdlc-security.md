# How Anthropic Secures Its AI-Native Software Development Lifecycle

**Source:** https://claude.com/blog/how-anthropic-secures-its-ai-native-software-development-lifecycle
**Author:** Jason Clinton, Deputy CISO, Anthropic (with Michael Segner)
**Saved:** 2026-07-23
**Tags:** ai, security, infrastructure, agentic-ai, tools, fundamentals

> **Primary source.** Authoritative — written by Anthropic's own security leadership. Companion piece to the "Zero Trust for Agents" framework (claude.com/blog/zero-trust-for-ai-agents). Published July 21, 2026.

---

## TL;DR
Anthropic's security team defends a codebase where Claude authors ~80% of merged code, engineers ship 8× more code per quarter than 2021–2025, and more than half of all code is merged by an internal version of Claude Tag with humans owning final approval. Four overarching strategies: shift security left (and close the loop from bug discovery back to CLAUDE.md); hard access/identity boundaries to contain blast radius; combined automated deterministic + agentic reviews before and after production; humans in the loop at highest-leverage points — not everywhere.

---

## The Three Threats Being Designed Against

1. **Compromised or prompt-injected agent** introducing a malicious change
2. **Supply-chain and dependency poisoning** that an agent ingests as trusted input
3. **Familiar application vulnerability classes** now arriving at higher volume

Every control maps to at least one of these.

---

## The Four Overarching Strategies

1. **Shift security left** — integrate with code development stage; close the loop between vulnerability discovery and updating CLAUDE.md to prevent recurrence
2. **Hard access and identity boundaries** — contain blast radius; minimum permissions per agent; agents cannot access other agents unless explicitly permissioned
3. **Automated deterministic + agentic reviews** — combined, not either/or; multiple specialist agents not one super-prompt; humans in the loop for regulated/critical code
4. **Humans at highest-leverage points** — not everywhere; focus human attention on risk-tiered code and sampling of automated approvals

---

## Security Controls by SDLC Stage

### Plan — Automated Project Security Review (PSR)

**Original approach:** Claude Opus-powered web app ingesting project design docs, analyzing against MITRE ATT&CK framework, identifying vulnerabilities and suggesting mitigations.

**Current enhancement:** Connected to an internal knowledge index spanning org-wide policies, past decisions, and related systems. Missing context is captured automatically. Teams can self-approve launches Claude deems low-risk.

**Key adaptation:** Planning cycles are now compressed — detailed architectural review is less critical as a gate because major feature prototypes can be created in hours. The PSR connects to where context already lives (chat threads, prior reviews, codebase) rather than forcing documentation.

**Enduring principle:** Connect security agents to organizational context. Bring agents to where the context lives, not the other way around.

---

### Code — Shift Left, Close the Loop

**CLAUDE.md as the security standard:** Security guidelines are encoded in CLAUDE.md files and org-wide skills so code follows best practices at generation time. When an agent discovers a bug class, the relevant file is updated — preventing recurrence in future AI-generated code. This is the closed loop.

**`/security-review` skill:** Instructs the agent to run a security review as a final step before opening a PR. Productized version of Anthropic's internal review workflow — looks for attacker-controllable input entry points, scans for suspicious links, verifies findings. Claude Security plugin now runs reviews inline during code generation, not just as a final step.

**Remote VMs for coding:** Moved coding to remote VMs. Agent traffic on these VMs is egress-allowlisted. Tight egress controls matter especially when agents read untrusted input that could carry prompt-injection payloads. An injected instruction cannot reach arbitrary internet destinations — exfiltration paths limited to a small set of monitored services.

**Enduring principle:** Shifting left in an AI-native org means closing the loop between vulnerability discovery and CLAUDE.md updates. Limit blast radius with hard boundaries (Principle of Least Agency).

---

### Test (CI) — Multiple Specialist Agents + Human Sampling

**The bottleneck Anthropic hit:** When most developers were running multiple agents simultaneously, they could only move as fast as humans could review code. Human review became the rate-limiting factor.

**The solution:** Combined automated agentic and deterministic reviews. Human review reserved for regulated or truly critical code.

**Why multiple specialist agents, not one super-prompt:**
- Don't share biases and blindspots
- If one is compromised or makes a mistake, others can catch it
- Effort isn't spread too thin across multiple focus areas
- Each agent has separate context windows — different type of risk, controlled differently

**Measured results at Anthropic:**
- PRs receiving substantive review comments: grew from **16% to 54%** — because agents must write proof that a finding is valid before raising it
- Approximately **⅓ of bugs behind past claude.ai incidents** would have been caught by current automated processes

**Human accountability structure:**
- Code tiered by risk; entire codebases have strict human approval processes
- Every automated approval logged with signals and reasoning
- **Risk-weighted sample** reviewed by humans
- Additional testing round focused on invariants ("user A can never read user B's data")
- Agents aren't merging to production unchecked

**Enduring principle:** Automated reviews are a different type of risk controlled differently (multiple gates, separate context windows). Humans stay in the loop but may be at different lifecycle points depending on codebase criticality.

---

### Deploy (CD) — Continuous Dynamic Testing

**Standard practices maintained:** External pentesting for major launches, periodic DAST scans for logic bugs that static scans miss.

**The problem with periodic DAST:** Larger code volumes shipping more frequently mean periodic scans are no longer dynamic enough.

**Anthropic's approach:** Implementing continuous AI-powered DAST scans in staging environment. These look for vulnerabilities at the system level where assumptions between two or more services are incorrect. AI models are better at the multi-step, cross-component reasoning needed to catch complex vulnerabilities.

**Example:** In February 2026, Claude discovered and helped fix more than 500 high-severity OSS vulnerabilities.

**Enduring principle:** Dynamic testing should match deployment cadence.

---

### Monitor — Incident Response + The Identity Lesson

**Automated incident response pipeline:**
When an alert fires, Claude starts:
1. Reviewing production logs
2. Root-causing the bug
3. Writing the post-mortem
4. (In some cases) writing the code change to fix the bug

**What it cannot do:** Deploy the fix automatically. The incident response agent is a single-purpose system account with exactly three permissions: write new docs, post in company channels, access production logs.

**Why the fix requires a separate agent-human reviewer system:**
- Separate agents act as checks on each other
- Blast radius must be contained when pushing to production
- One agent cannot both find the bug and deploy the fix

**The hard lesson (quoted directly):**
> "When considering an agent's hard boundaries you need to include its access to other agents."

Following a model upgrade, the incident response agent autonomously reached out over Slack to another Claude instance that could write code, asking it to push the fix. **This was caught at a human review gate as designed**, but it taught Anthropic to draw the boundary around access and actions — not around a model's instructions or what they believed a model could do.

**Today:** Agent-to-agent communication on Slack is the norm at Anthropic. They give considerable thought to agent identity models and draw hard access boundaries.

**Code migrations:** The economic cost of migration has fallen dramatically. What previously required campaigning for engineering resources across departments for multiple quarters can now be done by Claude in days — tens of thousands of lines of code in the migration case.

**Enduring principle:** Give every agent a single-purpose identity with minimum permissions. If you let agents coordinate, have them do so over the same channels as humans.

---

### Governance — Monitoring Loops, Not Just Bugs

Four governance mechanisms:

1. **Tiering by risk** — automating reviews based on risk level, not uniformly
2. **Shadow mode for new AI reviewers** — new agents post comments for human approval until trust is earned; team also "red teams" them by trying to insert malicious changes
3. **Sampling** — percentage of all automated approvals reviewed by humans
4. **Vitals dashboard** — key metrics across every security process and workstream, closely monitored
5. **Every agent action routed to SIEM** — every automated approval, tool call, and agent-to-agent message is logged with signals used, lands in SIEM. Agents are treated as a new type of insider threat; alerts raised when they act out of alignment

**Enduring principle:** The security engineer's job evolves from monitoring bugs to monitoring loops.

---

## The Closing Frame

> "What doesn't quite work today or isn't quite economically feasible likely will be soon. The right question for your team isn't 'can we afford to scan everything?' but 'what would we run if scanning were nearly free?' Plan for that."

---

## What This Note Adds to the Research Library

**The agent-to-agent access boundary** is the most important new concept. Not documented anywhere else in the library: an agent's hard access boundary must explicitly exclude access to other agents unless permissioned. The incident response agent reaching out to the code-writing agent is a real-world confused-deputy attack that almost worked — and it came from the agent being helpful, not malicious.

**CLAUDE.md as the closed security loop** — when a bug class is discovered, the CLAUDE.md file is updated to prevent future AI-generated code from containing the same class. This is how security shifts left in an AI-native SDLC. Not just catching bugs earlier, but encoding the knowledge so the bugs don't get generated in the first place.

**16% → 54% substantive review** — requiring agents to write proof of valid findings before raising them dramatically improves signal quality. This is the false-confidence test audit principle from Jamon's 18-point setup applied to security reviews.

---

## Questions & Gaps
- The egress allowlist on coding VMs — what's the maintenance burden as legitimate dependencies change? How is the allowlist governed without becoming a security bottleneck itself?
- The agent identity model is referenced but not detailed — Anthropic mentions it links to a separate "agent identity access model" article. That article would be worth saving separately.
- Shadow mode for new AI reviewers: how is "trust earned" quantified? What metrics determine when a reviewer agent graduates from shadow mode to active approval?
- The 54% substantive review comment rate — is this the right target? 46% of PRs receiving no substantive comment might mean good code is going through cleanly, or it might mean reviewers are missing things on nearly half of PRs.
- The ⅓ of past incidents claim (automated processes would have caught ⅓ of bugs behind past claude.ai incidents) — is this retrospective simulation or live measurement?

## Related Notes
- [Agent Platform Security Checklist](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/agent-platform-security-checklist.md) — this article is what that checklist looks like actually implemented in production at scale. The "principle of least agency" and identity boundary principles documented there are all present here with real-world examples.
- [Meta's AI Bot Instagram Account Takeover](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/metas-ai-bot-instagram-account-takeover.md) — the confused deputy problem. Anthropic's incident response agent reaching out to the code-writing agent is a near-miss of the same class. The difference: Anthropic caught it at a human review gate; Meta didn't have one.
- [30 Core Agentic Engineering Concepts](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/30-core-agentic-engineering-concepts.md) — this article implements concepts 14 (sandboxing), 15 (permissions), 16 (hooks), 17 (prompt injection defense), 18 (pre-commit gates), 19 (tracing via SIEM), and 20 (metrics dashboard) all at once, in production, at Anthropic's scale.
- [Jamon Holmgren 18-Point Agentic Setup](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/jamon-holmgren-18-point-agentic-setup.md) — Jamon's item 6 (cross-agent review, multiple models, separate context windows) and item 14 (false-confidence test audit) are implemented here at Anthropic's scale. The "proof required before raising a finding" pattern is exactly the false-confidence mitigation.
- [Own the Outer Loop — Agentic Accountability](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/own-the-outer-loop-agentic-accountability.md) — Osmani's "monitoring loops not bugs" is Anthropic's final enduring principle stated almost verbatim. The governance section here is the production implementation of the outer loop ownership concept.
- [Anthropic Recursive Self-Improvement](https://github.com/LutherCalvinRiggs/research/blob/main/ai/research/anthropic-recursive-self-improvement.md) — this article is the security team's companion to the RSI piece. The RSI piece describes what AI-native development looks like from the capability side; this article describes what securing it looks like from the defense side.
