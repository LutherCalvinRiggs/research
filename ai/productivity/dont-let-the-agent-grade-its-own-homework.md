# Don't Let the Agent Grade its Own Homework

**Source:** https://jedarden.com/notes/dont-let-the-agent-grade-its-own-homework/
**Saved:** 2026-06-03
**Tags:** ai, prompting, productivity, agentic-ai, orchestration

---

## TL;DR
An agent reporting "done" is the actor certifying its own work. It cannot catch failures caused by its own misunderstanding. Verification must come from somewhere the work didn't — the live artifact, a separate agent, or a different tool. Interactively you spawn the watcher by hand; in a fleet it's a validation gate the task must clear before closing.

## Key Concepts & Terms
- **Self-certification bias**: If the agent misunderstood the goal, its definition of "done" inherited the misunderstanding. If it set the wrong env var, its model of "running" excludes that var. Asking it to re-verify runs the check through the exact blind spot.
- **Independent observer**: Three sources: (1) The live artifact itself — query the pod status directly, don't ask the agent if the pod is running; (2) A separate fresh agent with no knowledge of how the work was done; (3) A different tool — compiler says it built, does the test suite agree?
- **Validation gate**: In a cattle fleet, a successful exit code is a claim, not a fact. The orchestrator holds a gate that the claim must pass before the task is allowed to close. The acceptance criteria written into each task are the "somewhere else" the check comes from.

## Main Arguments & Takeaways
- **"Done" is a press release, not a fact**: The agent reports what its internal process concluded, not what the external world shows. "The pod is deployed and running" — the pod was in CrashLoopBackOff. The agent had done all its steps correctly; it just never looked at the thing it built from outside.
- **The verifier must be able to disagree with the actor**: If your check can only ever return "yep, looks good," it isn't a check. It's a mirror.
- **Build verification into the original request, not as a follow-up**: "Build the image, confirm it builds, then update the deployment, then confirm the new pod is actually running." One message, not two. The agent's success report lowers your guard; baking the check in means verification happens while you're still paying attention.
- **Fleet translation**: Interactively you are the gate — cheap and reversible. In the state machine, a successful exit code is a claim that has to clear a validation gate before the task is allowed to close; if the gate fails the task releases back to the queue.
- **The rule is identical at both scales**: The thing that did the work does not get to be the thing that certifies it.

## Notable Quotes
> "Fluent confidence is sedating. By the time you've read 'deployed and running,' some part of you has already moved on."

## Questions & Gaps
- What does a practical independent-agent verification step look like in a NEEDLE-style fleet? Spawning a new worker whose only input is the artifact and the original acceptance criteria?
- The "different tool" check — how do you chain compiler → test suite → integration test as a structured validation pipeline?
- At fleet scale, how expensive is spawning a verification worker for every task? Is there a tiering approach (light/heavy validation based on task risk)?

## Related Notes
- [Deterministic State Machines for Non-Deterministic Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/deterministic-state-machines-for-agents.md) — the validation gate as an outcome in the state machine.
- [Anchor on a Fact the Model Can't See](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/anchor-on-a-fact-the-model-cant-see.md) — same verification instinct from the human's side.
- [Harness Design for Long-Running Apps](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/harness-design-for-long-running-apps.md) — Anthropic's generator-evaluator pattern is the same principle at the multi-agent architecture level.
