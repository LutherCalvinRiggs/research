# 'No' is Not an Instruction

**Source:** https://jedarden.com/notes/no-is-not-an-instruction/
**Saved:** 2026-06-03
**Tags:** ai, prompting, productivity, agentic-ai, learning

---

## TL;DR
A bare rejection tells the agent one bit of information: the last thing was wrong. The space of "not that" is enormous. A rejection that names the alternative re-steers in one turn; one without it buys another guess. Interactively this costs a turn; across a cattle fleet the same vague rejection in a plan costs N failed executions.

## Key Concepts & Terms
- **Content-free rejection**: "No, don't do that." / "Be more careful." / "That's not what I meant." These feel like instructions but contain no destination. The model's search space narrows by one excluded option; everything else remains.
- **The naming discipline**: "Don't X, do Y." Never send a rejection without the alternative in the same message. If you don't know the alternative yet, that's a different — and more honest — message: "stop, I need to think."
- **Asymmetry at scale**: Interactively, a vague rejection costs one extra turn. In a cattle fleet, it's information you didn't write into the plan — every worker that hits the ambiguity fails and retries before you notice the pattern.

## Main Arguments & Takeaways
- **The agent is walking a random path through "not what you wanted" until you name a target**: Audio transcription example — "don't use OpenAI" → agent tries AssemblyAI → "no" → tries Deepgram → three turns with zero forward progress.
- **Almost every time you reach for "no," you already know the answer**: The right thing is there, you just didn't type it because rejecting felt faster. It isn't. The half-second saved on the first message is paid back with interest on the next guess.
- **Mechanical rule**: Not allowed to send a rejection until it contains the alternative. If you genuinely don't know — don't masquerade as a correction.
- **Technically-a-redirect-but-points-nowhere**: "Do it properly." "Be more careful." "That's not what I meant." These are the soft version of the failure — they feel like instructions and contain none.
- **Fleet translation**: The interactive habit (name the alternative) becomes the written plan requirement (specify the constraint). A rejection you'd have typed becomes a written rule in the task bead. The discipline is identical; the surface is different.

## Questions & Gaps
- When you genuinely don't know the alternative yet, what's the best practice — pause the session, or continue with a holding statement?
- Are there domains where naming the alternative is genuinely hard (e.g. aesthetic/design judgments)? How do you handle those?
- The "one bit of information" framing — is this approximately right, or are there cases where context-rich rejections narrow the space more dramatically?

## Related Notes
- [The Plan is the Prompt](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/the-plan-is-the-prompt.md) — at plan level, the same principle: unspecified constraints = gaps that cost N workers.
- [Pet Agents vs. Cattle Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/pet-agents-vs-cattle-agents.md) — the scaling context that makes this asymmetry visible.
