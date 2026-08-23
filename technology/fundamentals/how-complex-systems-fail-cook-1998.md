# How Complex Systems Fail

**Source:** https://how.complexsystems.fail/
**Author:** Richard I. Cook, MD — Cognitive Technologies Laboratory, University of Chicago
**Original:** 1998, 1999, 2000
**Saved:** 2026-08-19
**Tags:** technology, infrastructure, fundamentals, psychology, philosophy

> Foundational text in safety engineering and systems thinking. Originally written about healthcare patient safety, but applies universally to any complex system — software infrastructure, air traffic control, power grids, and AI agent fleets. 18 propositions. Approximately 2,500 words. Widely cited in software reliability engineering and Site Reliability Engineering (SRE) literature.

---

## TL;DR
18 propositions about why complex systems fail. Key throughline: catastrophe requires multiple simultaneous failures, not a single root cause; complex systems always run in degraded mode with latent failures present; "root cause" attribution is a social construct, not a technical truth; humans are not the cause of failure but the primary defense against it; and safety is an emergent property of the system, not a property of any component. Written about hospitals; applies everywhere.

---

## The 18 Propositions

### 1. Complex systems are intrinsically hazardous systems
The hazard is inherent to the nature of the system, not an anomaly. You cannot build a complex system that is fundamentally safe — you can only build defenses against the intrinsic hazards.

### 2. Complex systems are heavily and successfully defended against failure
Multiple layers: technical (backups, safety features), human (training, knowledge), organizational (policies, procedures, certification). These defenses normally work. Operations are generally successful. This is the baseline.

### 3. Catastrophe requires multiple failures — single point failures are not enough
The defenses work. Overt catastrophic failure occurs when small, apparently innocuous failures combine to create an opportunity for systemic accident. **Each failure is necessary but only the combination is sufficient.** There are many more failure opportunities than overt accidents. Most trajectories are blocked — by design, or by practitioners.

### 4. Complex systems contain changing mixtures of failures latent within them
It is impossible to run a complex system without multiple flaws being present. They are individually insufficient to cause failure, so they're treated as minor factors. Eradication is economically limited, and it's hard to see before the fact how they might combine. The failures change constantly because technology, work organization, and eradication efforts change.

### 5. Complex systems run in degraded mode
A corollary: complex systems run as *broken* systems. They function because of redundancies and because people make them function despite the flaws. Post-accident reviews almost always note a history of prior "proto-accidents" that nearly generated catastrophe. Arguments that degraded conditions "should have been recognized" before an accident are predicated on naïve notions of system performance.

### 6. Catastrophe is always just around the corner
The potential for catastrophic failure is always present. It is impossible to eliminate it — it is inherent to the system's nature. Practitioners are always in close proximity to these potential failures. Disaster can occur at any time and in nearly any place.

### 7. Post-accident attribution to a "root cause" is fundamentally wrong
Because overt failure requires multiple faults, there is no isolated "cause." There are multiple contributors, each necessarily insufficient alone. The concept of "root cause" does not reflect a technical understanding of failure — it reflects a **social and cultural need to blame specific, localized forces or events** for outcomes. The isolation of a root cause is not technically possible.

### 8. Hindsight biases post-accident assessments of human performance
Knowledge of the outcome makes the events leading to it seem more salient than they were at the time. It seems that practitioners "should have known" the factors would "inevitably" lead to an accident. **Hindsight bias remains the primary obstacle to accident investigation, especially when expert human performance is involved.** This is not specific to medical or technical judgments — it is a feature of all human cognition about past events.

### 9. Human operators have dual roles: as producers AND as defenders against failure
Practitioners operate the system to produce its desired product AND work to forestall accidents. This is unavoidable. Outsiders rarely acknowledge both roles. In non-accident times: the production role is emphasized. After accidents: the defense role is emphasized. At either time, the outsider's view misapprehends the operator's constant, simultaneous engagement with both.

### 10. All practitioner actions are gambles
After accidents, actions appear as "blunders" or "deliberate disregard." But all practitioner actions are gambles — acts taken in the face of uncertain outcomes. The degree of uncertainty changes moment to moment. The converse — that **successful outcomes are also the result of gambles** — is not widely appreciated.

### 11. Actions at the sharp end resolve all ambiguity
Organizations are often deliberately ambiguous about the relationship between production targets, cost efficiency, and acceptable risk. All ambiguity is resolved by actions of practitioners at the sharp end. After an accident, these actions may be labeled "errors" or "violations" — but these evaluations ignore the driving forces, especially production pressure.

### 12. Human practitioners are the adaptable element of complex systems
Practitioners and first-line management actively adapt the system to maximize production and minimize accidents, often moment to moment. These adaptations include:
- Restructuring to reduce exposure of vulnerable parts to failure
- Concentrating critical resources in areas of expected high demand
- Providing pathways for retreat or recovery from faults
- Establishing early detection of changed system performance to allow graceful cutbacks

### 13. Human expertise in complex systems is constantly changing
Complex systems require substantial human expertise. Expertise changes as technology changes, and because experts leave and must be replaced. At any moment, a given system will contain practitioners and trainees with varying degrees of expertise. Critical issues: using scarce expertise where most needed, and developing expertise for future use.

### 14. Change introduces new forms of failure
The low rate of overt accidents in reliable systems may encourage changes — especially new technology — to decrease low-consequence, high-frequency failures. These changes may actually create opportunities for new, low-frequency but high-consequence failures. The new, rare catastrophes can have greater impact than the ones eliminated. These new failure forms are difficult to see before the fact. Because they occur at low rate, multiple system changes may occur before an accident, making it hard to attribute causation.

### 15. Views of "cause" limit the effectiveness of defenses against future events
Post-accident remedies for "human error" typically obstruct activities that "caused" the accident. These end-of-chain measures do little to reduce the likelihood of further accidents. The likelihood of an *identical* accident is already extraordinarily low because the pattern of latent failures changes constantly. Instead of increasing safety, post-accident remedies usually **increase coupling and complexity**, creating more potential latent failures and making detection of accident trajectories harder.

### 16. Safety is a characteristic of systems, not of their components
Safety is an **emergent property** of systems. It does not reside in a person, device, or department. Safety cannot be purchased or manufactured as a separate feature. The state of safety in any system is always dynamic; continuous systemic change ensures hazard and its management are constantly changing.

### 17. People continuously create safety
Failure-free operations are the result of activities of people who work to keep the system within the boundaries of tolerable performance. System operations are never trouble-free — human adaptations to changing conditions **create safety from moment to moment**. These adaptations are sometimes well-rehearsed routines; sometimes novel combinations; sometimes de novo new approaches.

### 18. Failure-free operations require experience with failure
Recognizing hazard and staying inside tolerable performance boundaries requires intimate contact with failure. More robust system performance arises in systems where operators can discern the "edge of the envelope." In intrinsically hazardous systems, operators are expected to encounter and appreciate hazards in ways that lead to overall desirable performance. Improved safety depends on providing operators with calibrated views of hazards and how their actions move performance toward or away from the edge.

---

## The Core Insights (Synthesized)

**On causation:** There is no single root cause of a complex system failure. Accidents are the product of multiple simultaneous latent failures that combine. "Root cause" is a social construct that satisfies the human need to assign blame, not a technical description of what happened.

**On the system's normal state:** Complex systems run in degraded mode all the time. The absence of overt accidents does not mean the system is safe — it means the system's defenses and human practitioners are successfully blocking failure trajectories. Every system is always one configuration of latent failures away from catastrophe.

**On humans:** Humans are not the cause of failure in complex systems. Humans are the primary defense against failure. The "human error" framing inverts the actual relationship. Practitioners create safety continuously, through constant adaptation, while also maintaining production. Their actions are gambles made under uncertainty — not negligence.

**On interventions:** Post-accident interventions that add rules, processes, and constraints typically increase coupling and complexity, which creates more latent failure opportunities rather than fewer. The right response to accidents is not adding more defenses — it is improving the system's ability to detect failure trajectories early and provide graceful degradation paths.

**On safety:** Safety is not a feature you add to a system. It is an emergent property of the system as a whole, created continuously by people adapting to conditions.

---

## Applying This Framework to Software Infrastructure

The GitHub outage of August 17, 2026 (saved in `github-outage-aug-2026-cascade-failure.md`) is a perfect textbook case:

**Proposition 3 (multiple failures required):** The outage required: misconfigured Istio autoscaling policy + new traffic peak + optimistic retry logic in the gateway + VS Code's retry behavior. No single one causes the outage.

**Proposition 4 (latent failures always present):** The misconfigured autoscaling policy was a latent failure that existed before the incident. The VS Code retry bug was a latent failure. They were not individually causing problems until they combined.

**Proposition 5 (degraded mode):** GitHub likely had many other latent failures present that day that did not combine in problematic ways. The system "ran" fine until this particular configuration of latent failures aligned.

**Proposition 7 (no root cause):** GitHub's post-mortem identifies "misconfigured Istio autoscaling policy" as the cause, but this is technically inaccurate. Any of the contributing factors removed would have prevented the outage. The "root cause" framing is socially useful for the remediation follow-up list, not technically accurate.

**Proposition 14 (change introduces new failures):** Istio service mesh was presumably added to gain reliability benefits. It introduced a new failure mode (sidecar concurrency limits invisible to autoscaling policy) that didn't exist before.

**Proposition 15 (remedies increase complexity):** The follow-up actions (auditing Istio limits, reviewing retry behavior, improving monitoring) will add new systems and constraints — which per Cook's proposition, may introduce new latent failures while reducing the specific failure trajectory that was just experienced.

---

## Questions & Gaps
- Cook's framework was developed for healthcare (1998) and has been adopted by SRE/software communities. How well does the framework translate to AI agent systems specifically? NEEDLE workers and LLM agents have different failure modes (non-determinism, hallucination) that don't map cleanly to the "human practitioner" role.
- Proposition 15 argues that post-accident remedies typically increase coupling and complexity. Is there any empirical evidence for or against this in software systems specifically? The SRE literature seems to suggest that well-designed remedies do reduce accident rates — is Cook's claim overstated, or is software infrastructure an exception?
- The framework assumes human practitioners as the adaptive element (proposition 12). As human practitioners are replaced by AI agents, who plays this role? Agents are adaptive but in different ways — and their failure modes are different.

## Related Notes
- [GitHub Outage Aug 2026 — Cascade Failure Analysis](https://github.com/LutherCalvinRiggs/research/blob/main/technology/infrastructure/github-outage-aug-2026-cascade-failure.md) — a real-world case study in Cook's framework. Saved the day before this note. Every proposition in this paper maps onto that incident.
- [Cursor Git at Scale — Continuity](https://github.com/LutherCalvinRiggs/research/blob/main/technology/infrastructure/cursor-git-at-scale-continuity.md) — Continuity's design philosophy ("designed to always be correct when degraded, and always fast when healthy") is a direct application of Cook's framework: assume degraded mode is normal, design for graceful degradation rather than preventing all failure.
- [NEEDLE Production Safety Guide](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/NEEDLE-Production-Safety-Guide.md) — the five-layer defense model in NEEDLE's safety guide reflects propositions 2 and 3: multiple layers of defense, because catastrophe requires multiple failures to align.
- [Own the Outer Loop — Agentic Accountability](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/own-the-outer-loop-agentic-accountability.md) — Osmani's "answerability" concept reflects Cook's proposition 8 (hindsight bias): when something goes wrong with a long-horizon agent, you need to be able to reconstruct the decision chain. Without that, you're subject to the same hindsight bias that makes post-accident analysis inaccurate.
- [Deterministic State Machines for Non-Deterministic Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/deterministic-state-machines-for-agents.md) — the exhaustive outcome table is a formalization of proposition 12 (practitioners provide pathways for retreat or recovery from expected and unexpected faults). The outcome table is the designed recovery pathway; the deterministic shell is what allows graceful cutback.
