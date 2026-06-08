# 6 Main Types of Trading Bots on Polymarket Up/Down Markets

**Source:** https://x.com/retrovalix/status/2055653020948484422
**Saved:** 2026-06-07
**Tags:** finance, research, economics, tools, crypto, trading

---

## TL;DR
Analysis of 1,000 profitable bots on Polymarket's short crypto Up/Down markets reveals six distinct strategy types. None of them "guess direction" — all six exploit market microstructure: repricing delays, order book imbalance, arbitrage between Up and Down, lag between 5-minute and 15-minute contracts, and price gaps in the final seconds before resolution.

## Key Concepts & Terms
- **Up/Down market**: A Polymarket binary contract that resolves based on whether a crypto asset (BTC, ETH) closes above or below its opening price over a fixed window (5 min, 15 min). One side pays $1 at resolution; the other pays $0.
- **Market microstructure**: The mechanics of how prices form and change — order book depth, bid/ask spread, repricing lag, liquidity gaps. Profitable bots trade microstructure, not macro direction.
- **Repricing lag**: The time gap between when the underlying asset (BTC price) moves and when Polymarket's contract price fully reflects that move. The primary source of edge for most bot types.
- **Limit orders**: Orders placed at a specific price, not executed at market. All six bot types use limit orders almost exclusively — with 1–5% edge, market order slippage would eliminate profit entirely.
- **EV (Expected Value)**: The probability-weighted average outcome. Profitable bots build positions with positive EV by exploiting structural mispricing, not by predicting direction.
- **Tail risk**: The risk of a low-probability but catastrophic outcome. Especially relevant to the Near-Resolution Bot — a last-second asset reversal wipes out many small wins.
- **Position structure**: How a bot constructs its exposure — e.g. buying both sides in different ratios, using one side as a hedge while the other is the main directional position.

## The Six Bot Types

### 1. Pure Arbitrage Bot
Buys both Up and Down when their combined price is below $1.00. Since one side must pay $1 at resolution, any combined entry below $1.00 locks in riskless profit regardless of outcome.

**Example:** Up = $0.45, Down = $0.46 → combined cost $0.91 → guaranteed $1.00 payout → $0.09 edge per dollar deployed.

**Why it's hard in practice:** These windows are brief. The bot must detect imbalance, place limit orders accurately, and execute before the market corrects. Poor execution destroys the edge.

**Core mechanics:** Both sides bought simultaneously · limit orders only · profit from price structure, not prediction · repeated small edge at scale.

### 2. Directional Arbitrage Bot
Starts from the same arbitrage structure (Up + Down < $1.00) but tilts the position toward whichever side its model identifies as undervalued. The weaker side becomes a partial hedge rather than a full hedge.

**Example:** Up + Down can be built for $0.96, but the model also shows Up is undervalued. Bot buys more Up and less Down — arbitrage base, directional upside.

**Why it works:** Pure arbitrage caps upside. The directional tilt adds a second profit source. Arbitrage becomes a protective framework rather than the final strategy.

**Core mechanics:** Arbitrage structure as foundation · directional tilt on the stronger side · second side as partial hedge · limit orders only · improved EV through position construction.

### 3. Repricing / Fair Value Model Bot
Builds an independent estimate of what each side's probability should be given the current underlying asset price, then compares it to Polymarket's actual price. Buys whichever side is trading below fair value.

**Why it works:** BTC price moves first. Polymarket price moves second. Between those two events is a time window. The bot that recalculates fair probability faster than the market can buy the undervalued side before the price corrects.

**Core mechanics:** Real-time fair value model · compares underlying asset price to contract price · buys undervalued side · limit orders · can add arbitrage as an additional hedge layer.

### 4. Cross-Timeframe / Multi-Market Bot
Trades several related contracts simultaneously — e.g. both the 5-minute and 15-minute BTC Up/Down markets. When BTC moves, both should reprice, but liquidity and trader activity differ between contracts, so one often reprices before the other. The bot buys the lagging contract.

**Why it works:** The same underlying price drives multiple timeframes, but they don't update synchronously. The 5-minute market may already reflect new reality while the 15-minute market still trades on old structure, or vice versa.

**Core mechanics:** Multiple related contracts monitored simultaneously · lag between timeframes identified · hedge through neighboring markets · limit orders only.

### 5. Order Book Imbalance Bot
Looks for structural imbalance — price skew between sides, uneven repricing, weak order book depth, mispricing between related markets. Does not require Up + Down < $1.00; instead it identifies when one side is temporarily undervalued relative to the overall position structure.

**Why it works:** Price alone doesn't show the full picture. What matters is where the imbalance appears, which side is structurally stronger, how related markets are moving, and whether a position can be built at better-than-fair-value through a two-sided structure.

**Core mechanics:** Price imbalance and short-term repricing · position built in parts · two-sided structure · trades related markets when mispricing appears · limit orders.

### 6. Near-Resolution Bot
Waits until the final seconds before market resolution, when the outcome is already nearly certain. The winning side may still trade at $0.98 or $0.99 rather than $1.00. The bot buys the near-certain side and waits for settlement.

**Why it works:** Polymarket doesn't instantly move almost-resolved outcomes to $1.00. For a human, 1–2% return in seconds looks trivial. For a bot repeating this hundreds of times, it becomes a significant strategy.

**Primary risk:** Tail risk. A last-second reversal in the underlying asset on a position sized for near-certainty can wipe out many prior wins in one trade.

**Core mechanics:** Buys outcomes close to resolution · enters in final seconds · targets prices around $0.98–0.99 · limit orders · high win rate with tail risk exposure.

## What All Six Have in Common

**1. Limit orders only.** Without limit order discipline, execution cost destroys edge. If a strategy captures 1–3% per trade, market order slippage makes it unprofitable.

**2. Small repeatable edge, not big predictions.** No bot is trying to call the direction of Bitcoin. Each is looking for many small situations where the price is temporarily wrong. The edge per trade is small; the number of trades makes it meaningful.

**3. Structure over direction.** The profitable question isn't "Will Bitcoin go up?" It's:
- Where is the price lagging behind reality?
- Where does Up + Down create positive EV?
- Where is the order book temporarily weak?
- Which timeframe hasn't repriced yet?
- Can the position be built cheaper than fair value?
- Can one side serve as a hedge while the other is the main position?

**4. The unified source of edge — lag.** Every strategy exploits a different version of the same underlying phenomenon:

| Bot type | Lag being exploited |
|----------|-------------------|
| Pure Arbitrage | Lag inside Up + Down combined pricing |
| Directional Arbitrage | Same lag + directional model advantage |
| Repricing Bot | Lag between underlying asset price and Polymarket price |
| Cross-Timeframe Bot | Lag between 5-min and 15-min contract repricing |
| Imbalance Bot | Lag in order book structure |
| Near-Resolution Bot | Lag before final settlement price correction |

**5. Risk managed through position structure.** Profitable bots don't just buy one side and wait. They build exposure — hedging one side against the other, building in parts, using neighboring markets as risk controls, deciding at each step whether to close or hold the stronger side.

## Questions & Gaps
- What's the typical edge per trade for each bot type at current Polymarket liquidity levels? The article shows the patterns but not the magnitude of opportunity.
- How has the profitability of these strategies evolved as Polymarket has grown? More liquidity should tighten spreads and reduce repricing lag — making these strategies harder over time.
- The Near-Resolution Bot's tail risk is described but not quantified — what's the practical frequency of last-second reversals in 5-minute BTC markets?
- The analysis was done with Claude on 1,000 bots — what was the methodology? On-chain transaction pattern analysis? Wallet profiling? Understanding this matters for replicating the analysis.
- Are any of these strategies available as open-source implementations, or are they all proprietary?
- How do these strategies interact when multiple bots of the same type compete? Does saturation kill the edge, and if so, at what bot count?
