# The Math Behind $40M Extracted from Polymarket — Arbitrage Deep Dive

**Source:** https://x.com/0x_Discover/status/2061044198418031017
**Saved:** 2026-06-10
**Tags:** finance, research, economics, fundamentals, trading, tools

> ⚠️ Editorial note: The article ends with a Polymarket referral/airdrop pitch that is unrelated to the research content. Notes cover the mathematical and infrastructure content only.
> **Primary source:** arXiv:2508.03474 — "Unravelling the Probabilistic Forest: Arbitrage in Prediction Markets" (2025). Secondary: arXiv:1606.02825v2.

---

## TL;DR
A 2025 research paper documented $39,688,585 in guaranteed arbitrage profits extracted from Polymarket between April 2024–April 2025. The top extractor made $2M from 4,049 trades ($496 guaranteed per trade). The edge isn't speed — it's mathematical infrastructure: integer programming over exponentially large outcome spaces, Bregman projection for optimal trade sizing, Frank-Wolfe algorithm for tractable computation, and parallel CLOB execution within the same 2-second Polygon block.

## Key Concepts & Terms

- **Marginal polytope problem**: Prediction market arbitrage isn't about checking if YES + NO = $1. It's about satisfying logical dependencies across correlated markets. "Republicans win Pennsylvania by 5+" and "Trump wins Pennsylvania" are logically constrained — a violation in one creates arbitrage in the other that no simple addition check catches.
- **Bregman divergence**: The correct distance metric for probability-space markets. A price move from $0.50 to $0.60 has different information content than $0.05 to $0.15 even though both are 10 cents. Standard Euclidean distance is wrong here — logarithmic cost functions must be respected.
- **Arbitrage-free manifold**: The set of all valid price states that contain no arbitrage. Finding the closest valid market state to the current (mispriced) state is a projection problem — and that projection tells you the optimal trade.
- **Frank-Wolfe algorithm**: An iterative optimization method that makes Bregman projection tractable. Instead of enumerating all 2^n valid outcomes, it builds an active set one vertex at a time via sequential linear programs. After 100 iterations you're tracking 100 vertices, not 2^63.
- **Active set**: The growing set of known valid outcomes that Frank-Wolfe uses to approximate the full feasible space. Grows by one vertex per iteration.
- **CLOB (Central Limit Order Book)**: Polymarket's execution model. Unlike DEX atomic swaps, CLOB execution is sequential — buy leg fills, then price updates before sell leg fills, turning a guaranteed profit into a loss. The 2-second Polygon block window is the structural constraint everyone operates within.
- **Kelly criterion (modified)**: Position sizing formula. In this context modified for execution risk and order book depth, with a 50% cap on available depth to avoid moving the market against yourself.
- **Gurobi**: Commercial integer programming solver used in the optimization engine's Layer 2.

## The Scale of the Problem

For any market with n conditions, there are 2^n possible price combinations:
- 305 US election markets → 46,360 possible pairs to check
- NCAA 2010 tournament (63 games) → 2^63 = 9.2 × 10^18 possible outcomes
- Duke vs Cornell basketball (7 possible win-counts each, 14 conditions) → 2^14 = 16,384 combinations, reduced to 3 linear constraints by logical dependencies

**Research findings from 17,218 conditions examined:**
- 7,051 (41%) showed single-market arbitrage
- Median mispricing: $0.60 (markets were regularly wrong by 40%)
- Not edge cases — systematically, massively exploitable

## The Maximum Profit Formula

The guaranteed profit from any arbitrage trade equals the Bregman divergence between the current market state and its projection onto the arbitrage-free manifold:

```
π* = D_φ(p* || p_current)

where:
  p_current = current (mispriced) market prices
  p* = projection of p_current onto arbitrage-free set M
  D_φ = Bregman divergence with respect to φ (the market's cost function)
```

Every one of the top extractor's 4,049 trades was solving this optimization. The $2M was the consistent output of running it faster and more accurately than every other participant.

## The Frank-Wolfe Solution (Making 2^63 Tractable)

```
Initialize: active set S = {one known valid outcome}

Each iteration:
  1. Solve convex optimization over current active set S → current estimate p̂
  2. Solve one integer program to find best new vertex v*
  3. Add v* to S
  4. Check convergence gap → stop when gap < threshold (typically 50–150 iterations)

Result: after 100 iterations, tracking 100 vertices instead of 9.2 × 10^18
```

**Why it gets faster as the event unfolds:** As outcomes settle, the feasible set shrinks — fewer variables, tighter constraints, faster integer program solves.
- Early tournament: 10–30 second solve times
- Late tournament: under 5 seconds
- After 45 games settled in NCAA: first successful 30-minute projection completed
- Frank-Wolfe outperformed baseline by 38% on median security prices for remaining games

## The Production System Architecture

### Data Pipeline
- Real-time WebSocket to Polymarket CLOB API (live order book + trade feeds)
- Alchemy Polygon node querying `OrderFilled` contract events
- Research analyzed 86 million transactions — requires infrastructure, not scripts

### Dependency Detection
- DeepSeek-R1-Distill-Qwen-32B with prompt engineering to classify market pairs
- Input: two market descriptions → Output: JSON of valid outcome combinations
- 81.45% accuracy on complex multi-condition election markets
- Sufficient for filtering; requires manual verification before execution

### Optimization Engine (3 layers)
```
Layer 1: Fast linear programming relaxations (milliseconds) — obvious mispricings
Layer 2: Frank-Wolfe + Gurobi IP solver (1–30 seconds) — complex correlated markets
Layer 3: Execution validation against live order book before any order submits
```

### Execution Window
```
Block N-1:  System detects arbitrage, submits all legs within 30ms
Block N:    All legs confirm, arbitrage captured, profit locked
Block N+1:  Slower systems see the trade on-chain
Block N+2:  Copy-bots attempt to replicate — market has already corrected
```

The 2-second Polygon block time is not a bug — it's the structural feature everyone operates within. The edge is the 30 seconds of detection-to-submission before that window closes.

### Position Sizing
```python
# Modified Kelly criterion
kelly_fraction = (edge - fees) / variance
position_cap = min(kelly_fraction, 0.50 * order_book_depth)
# Recalculated in real-time based on current portfolio value
```

## Where the $39.7M Came From

Three distinct strategy types:
1. **Single-market arbitrage** — YES + NO < $1.00 within one market
2. **Cross-market logical dependency** — violations between correlated markets (the marginal polytope problem)
3. **Temporal/settlement arbitrage** — near-resolution pricing gaps (the Near-Resolution Bot pattern from the other Polymarket note)

**Distribution:**
- Top 10 extractors: $8,127,849 (20.5% of total)
- Top single extractor: $2,009,632 from 4,049 trades ($496/trade average)
- 365 days of systematic operation

## The 15 Documented Profitable Wallets (Publicly Visible On-Chain)

| Wallet | Strategy | Documented Profit |
|--------|----------|------------------|
| kch123 | Latency arb, high frequency | $12,000,000 |
| RN1 | Market making, multi-market | $7,400,000 |
| Swisstony | Oracle arb, Chainlink | $5,900,000 |
| GamblingIsAllYouNeed | News-driven, AI probability | $4,600,000 |
| DrPufferfish | Combinatorial arb, multi-market | $3,400,000 |
| sovereign2013 | Latency arb, BTC/ETH contracts | $3,400,000 |
| 0x2a2C53bD27 | Market rebalancing, systematic | $2,500,000 |
| Countryside | Election markets, base rate | $2,400,000 |
| gatorr | Latency arb, parallel execution | $2,300,000 |
| weflyhigh | Multi-strategy, diversified | $1,800,000 |

Total across 15 wallets: >$51M documented profit.

## The Copy-Trading Trap

```
What traders think:
  Block N: profitable wallet executes
  Block N+1: you copy the position → same profit

What actually happens:
  Block N-1: fast system detects, submits in 30ms
  Block N: all legs confirm, arbitrage captured at $0.322
  Block N+1: you see it and copy
  Block N+1: you pay $0.344 for what they bought at $0.322
  → You are providing exit liquidity, not arbitraging
```

The only way copy-trading works is if your execution fires in the same block as the original. That's a hard technical requirement, not a preference.

## Questions & Gaps
- The 81.45% accuracy on AI-powered dependency detection — what's the cost of the 18.55% miss rate in practice? A false positive that triggers execution on a spurious dependency could be expensive.
- The research covers April 2024–April 2025 — how much has the landscape changed since? As more sophisticated actors enter, does the median mispricing shrink?
- The Frank-Wolfe solve time (1–30 seconds for complex cases) — is this fast enough for fast-moving markets, or is it primarily useful for longer-duration markets where prices don't move that quickly?
- DeepSeek-R1-Distill-Qwen-32B for dependency classification — what does this cost at scale? Running LLM inference on every new market pair is non-trivial.
- The 50% order book depth cap for position sizing — what happens in thin markets where even 50% depth moves prices significantly?

## Related Notes
- [6 Main Types of Trading Bots on Polymarket](https://github.com/LutherCalvinRiggs/research/blob/main/finance/research/polymarket-6-trading-bot-types.md) — the Near-Resolution Bot and Pure Arbitrage Bot described there are the consumer-facing versions of the strategies documented here. The "mathematical infrastructure" gap between retail and quant is exactly the gap this note documents in detail.
