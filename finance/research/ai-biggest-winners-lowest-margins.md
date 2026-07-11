# AI's Biggest Winners Have the Lowest Margins

**Source:** Daniel Kornum (publication/URL not confirmed — notes based on pasted excerpt)
**Saved:** 2026-07-11
**Tags:** finance, business, economics, fundamentals, technology

> ⚠️ Partial content — Sections 1–4 (Step 1) captured. Still missing: Steps 2 and 3 of the three-step framework, provider architecture, specific company examples.

---

## TL;DR
The biggest AI winners may not be software companies or AI labs — they may be low-margin businesses like manufacturers, trucking carriers, distributors, staffing agencies, and field-service operators that have run on single-digit margins for decades. For a business at 30% margins, AI efficiency gains barely move the needle. For a business at 3% margins, a sub-1% cost reduction can produce a 25%+ profit increase. AI gives these businesses a way to attack coordination costs (scheduling, dispatch, approvals, exception handling — ~6% of revenue, ~25% of labour spend) that were previously treated as permanent structural constraints — and early movers bank the earnings uplift before competitors force it back into lower prices.

---

## The Core Thesis

The traditional framing of AI value creation focuses on three levers: revenue (better products, faster sales, more productive employees), cost, and risk. Most attention has gone to the revenue lever.

But for low-margin businesses, the biggest lever is cost — and the math is asymmetric:

| Business type | Current margin | Cost reduction | Profit increase |
|---------------|---------------|----------------|-----------------|
| Software company | 30% | 1% | ~3% |
| Low-margin operator | 3% | <1% | >25% |

The same absolute cost reduction produces an order-of-magnitude larger proportional profit impact at thin margins. This is why the framing "AI companies are the winners" may miss the most important value creation story.

---

## Why Low-Margin Businesses Have Been Structurally Trapped

Low-margin industries — manufacturing, trucking, distribution, staffing, field services — have historically operated in environments where:

- Costs were treated as permanent structural constraints, not optimization problems
- Coordination overhead (scheduling, routing, dispatch, matching) was expensive and manual
- Thin margins left no budget for technology transformation
- Competitors faced the same constraints, so no single player could pull away

AI changes the equation on the coordination cost problem specifically. These businesses are not low-margin because they're inefficient at their core operations — they're low-margin because coordination at scale is expensive. AI attacks coordination costs directly.

---

## The First-Mover Advantage

Efficiency gains from AI eventually spread across commoditized markets — competitors adopt the same tools, prices adjust, and the margin improvement gets competed away. But the early movers in each industry:

1. Bank the earnings uplift while competitors haven't yet moved
2. Reset their cost position ahead of the field
3. Use the margin improvement to fund further advantages (talent, capital, pricing flexibility)

The window between "first mover captures the gain" and "efficiency spreads to the whole industry" is the investment opportunity. The article suggests this window exists right now in industries that have been overlooked precisely because they're not "AI companies."

---

## The Provider Opportunity

"The providers who solve this will build billion-dollar companies — and the businesses they transform will be the ones that escape the margin trap first."

This implies a two-sided opportunity:
- **Operators**: Low-margin businesses that move first capture structural margin improvement
- **Providers**: B2B AI companies that solve the coordination cost problem for these industries — not general-purpose AI tools, but vertical solutions built specifically for the coordination workflows of trucking, distribution, staffing, field services, etc.

---

## What the Article Doesn't Cover Yet (Gaps from Partial Content)

The excerpt ends at the end of Section 1. Missing sections likely include:

- **The coordination cost problem in depth** — what coordination costs look like in each industry, why they've been hard to attack historically
- **Specific industry examples** — how AI attacks coordination costs in trucking vs. staffing vs. field services vs. manufacturing
- **The "providers who solve this"** framing — what the right product architecture looks like for vertical AI in low-margin industries
- **Risk and implementation challenges** — why most low-margin businesses haven't moved yet (budget constraints, tech debt, change management)


---

## Section 2: The Structural Barriers (and the Coordination Cost Opportunity)

### Why Low-Margin Businesses Are Trapped

Three interlocking constraints keep these companies structurally thin:

1. **Commoditized markets** — they cannot move the market price; the market sets it. Cost is the only lever they control.
2. **Limited pricing power** — any efficiency gain that spreads to competitors gets competed away into lower prices for buyers.
3. **Large, sticky operating cost bases** — previously impossible to reduce without hurting service quality.

### The Labour Cost Anatomy

In labour-intensive, low-margin businesses:

| Cost layer | % of revenue |
|-----------|-------------|
| Total labour cost | ~25% of revenue |
| Of which: coordination labour (managing, coordinating, administering) | ~6% of revenue (~25% of labour spend) |

The physical work itself is one cost. The cost of *coordinating* that work — scheduling, dispatching, approvals, exception handling, administrative loops — is a separate, largely hidden burden on top.

### The Coordination Cost Thesis (The Precise Opportunity)

Coordination work is where AI has the clearest opportunity:

- Scheduling and dispatch
- Approval workflows
- Exception handling
- Administrative loops

These are not the physical jobs. They are the management layer around the physical jobs — and they consume roughly a quarter of the total labour budget.

### The Precise Math

```
Company operating at 3% net margin
Labour = 25% of revenue
Coordination labour = 6% of revenue (25% of labour spend)

Ease coordination burden by 10%:
  → Save 0.6% of revenue in coordination costs
  → On a 3% margin base: +0.6% / 3% = +20% earnings improvement
```

A 10% reduction in coordination overhead — not total labour, just the coordination fraction — produces a 20% earnings improvement for a 3% margin business. This is not incremental optimization. It restructures the entire earnings profile.

### The Structural Advantage Window

The first-mover dynamic in more detail:

- Early adopters capture the coordination cost reduction as margin *before* competitors move
- When competitors adopt, the efficiency advantage gets competed into lower prices — the coordination saving becomes a customer subsidy
- But the early mover has already: (a) banked multiple years of elevated earnings, (b) used that capital to invest in further advantages, (c) reset their cost structure permanently below the pre-AI baseline

AI does not just make these businesses slightly more efficient. It gives early adopters "a chance to open a structural cost advantage over their competitors — and to run as a genuinely higher-margin business, perhaps for the first time."


---

## Section 3: The Adoption Paradox

The companies with the most to gain from AI are often the least able to adopt it.

### Why Standard Enterprise AI Fails Here

Most enterprise AI products are built on a flawed assumption: that employees will adopt a new tool, use it correctly, and gradually turn usage into P&L value. This assumption doesn't hold reliably even inside tech-forward software companies. Inside a manufacturing plant, a trucking operation, or a labour-heavy distribution centre, it fails completely.

**The workforce profile mismatch:**
- These businesses employ workers who are not used to adopting new software products
- They are often the least susceptible to change management initiatives
- New "interaction surfaces" (dashboards, apps, copilots) require behaviour change that the organisation cannot reliably produce

### The Real Design Constraint

The question is not "how do we get employees to use AI?" The question is:

> **How do we get AI-driven margin expansion without relying on employee adoption — or at least without enforcing new interaction surfaces?**

This reframes the product design problem entirely. The solution is not a better copilot or a more intuitive dashboard. The solution is AI that works *around* the existing workforce rather than *through* it — running in the background, automating the coordination layer without requiring the frontline worker or even the middle manager to change their behaviour.

### The Market Opportunity

This adoption paradox — high potential gain, low adoption capability — creates the gap that defines the biggest unaddressed opportunity in enterprise AI. The article frames this as "the most addressable trillion-dollar opportunity in AI right now."

The providers who crack the adoption constraint for low-margin, labour-heavy businesses will not be building copilots. They will be building systems that:
- Intercept and automate coordination workflows at the infrastructure layer
- Produce margin improvement without requiring new behaviour from the existing workforce
- Deliver value in the P&L before any behaviour change occurs


---

## Section 4: Three Steps to Solve the Trillion-Dollar Challenge

### Step 1 — Find the Hidden Coordination Cost

**The narrow framing to avoid:** Most AI cost-saving conversations start with "replace a task, reduce headcount, or make an employee faster." That may happen eventually, but it misses the immediate, larger opportunity.

**The broader framing:** The opportunity is the *work behind the work* — the overhead required to keep messy human operations moving.

```
Frontline employee executes the task
                    ↕
Coordination layer keeps it moving:
  - Managers and supervisors
  - Operations teams
  - Dispatch and routing
  - Customer updates
  - Claims, invoices, exceptions
  - Back-office reconciliation
  - Finance and analytics teams
```

**Why the coordination layer is so large:** Human work is inherently messier than AI. Humans make judgment calls differently; each person carries their own context of the company and the task. The coordination layer exists to absorb that variability — to ensure work routes correctly, exceptions get handled, and information flows between people who each hold different pieces of the picture. Over time, this becomes a massive, load-bearing operating cost.

**Real example (logistics company):**
- Visible labour cost: the drivers
- Hidden coordination cost: dispatch teams, routing changes, customer updates, claims, invoices, exceptions, back-office reconciliation
- That coordination overhead: **~10% of revenue**
- That spend became the transformation target

**The pattern generalises across industries:**
- Logistics
- Manufacturing
- Facilities management
- Field services
- Staffing
- Healthcare clinics

In every case: labour-heavy, hard-to-differentiate service, limited pricing power, large coordination overhead keeping the operation reliable. The coordination layer is not a bug — it's what makes the service work. But it's also the cost that AI can attack without touching the frontline execution.

**The structural trap in one sentence:** These companies cannot raise prices to escape. Their margins stay compressed because they need a large coordination layer to deliver a commoditised service reliably — and until now, that coordination layer was not reducible without degrading service quality.

## Questions & Gaps
- Section 2 provides the precise math (10% coordination cost reduction → 20% earnings improvement at 3% margins). The next question is documented real-world examples — which specific companies have achieved this, and over what timeframe?
- Section 2 defines coordination costs as scheduling, dispatch, approvals, exception handling, and administrative loops — roughly 25% of labour spend / 6% of revenue. The remaining gap is industry-specific breakdown: what coordination looks like in trucking vs. staffing vs. field services.
- The first-mover window: how long does it typically last in industries this commoditized? Historical analogies from prior technology waves (GPS routing in trucking, ERPs in manufacturing) would be useful anchors.
- The provider framing raises a classic B2B dilemma: low-margin businesses are price-sensitive buyers. How do vertical AI providers price solutions such that the customer keeps enough of the margin improvement to justify the purchase?
- Which specific companies are already playing this game today? The article promises to name them but the excerpt ends before reaching that section.
- The "no new interaction surface" design constraint is the key product insight. What does this actually look like technically — API-layer integrations, workflow interception, RPA hybrids? Section 3 names the constraint but not the architecture.

## Related Notes
- [AI Company Analysis Forensic Screener](https://github.com/LutherCalvinRiggs/research/blob/main/finance/research/ai-company-analysis-forensic-screener.md) — the Piotroski F-Score (is this company getting financially stronger?) and Altman Z-Score (bankruptcy risk) are directly applicable to identifying which low-margin operators are executing the AI transformation vs. which are deteriorating.
- [Strategy Decay Monitoring Framework](https://github.com/LutherCalvinRiggs/research/blob/main/finance/research/strategy-decay-monitoring-framework.md) — the first-mover advantage argument here maps onto that framework's "crowding" concept: the margin uplift is real but temporary, decaying as competitors adopt the same tools. The monitoring question becomes: how do you detect when the efficiency window is closing?
- [Unit Economics of Running Cattle](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/unit-economics-of-cattle.md) — Jed's cost governance for agent fleets is the same discipline applied to the operator side: a low-margin business running AI agents needs the same spend caps and execution bounds, or the AI cost replaces the coordination cost rather than eliminating it.
- [Linear Regression Alpha Signals](https://github.com/LutherCalvinRiggs/research/blob/main/finance/research/linear-regression-alpha-signals-quant-framework.md) — the signal here is the margin expansion event: identifying which low-margin operators are in the first-mover window and trading that information before competitors catch up is a classic alpha signal problem. IC stability would test whether "early AI adopter in a thin-margin industry" is a reliable predictor of outperformance.
