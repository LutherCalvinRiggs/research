# How Top Quants Catch a Dying Strategy Before the Drawdown

**Source:** https://x.com/horizon_trade_x/status/2063992615721435154 (pasted content by @horizon_trade_x / Horizon)
**Saved:** 2026-06-11
**Tags:** finance, research, trading, fundamentals, economics, tools

---

## TL;DR
Every strategy has a half-life. In 2016, two finance professors tested 97 published trading strategies — after publication, the average edge dropped 58%. The framework for catching decay before it kills you: understand the three forces that kill edges, track four numbers weekly against backtest baselines, and write kill criteria before deploying capital — because the version of you that is down 12% will negotiate with rules you set when you were rational.

## Key Concepts & Terms
- **Performance cone**: A simulation constructed by resampling the historical trade list thousands of times, showing how deep drawdowns get and how long they last while the edge is intact. The live equity curve inside the cone = normal variance. Outside the cone = information. Every grey path in the visualization is the same strategy resampled — while live performance stays inside, nothing has happened yet.
- **Rolling Sharpe (90-day)**: The live rolling Sharpe compared to the backtest distribution. Below the 5th percentile is an early decay flag.
- **Time underwater**: How long the strategy has been below its previous high-water mark. "A strategy can die by going flat for two years without a deep drawdown" — duration is the underrated metric. Track against the longest backtest stretch, not just depth.
- **Trade-level drift**: Hit rate, profit factor, average win vs. average loss tracked over time. Decay typically appears as winners getting slightly smaller for 6 months in a row before the drawdown becomes visible.
- **Slippage drift**: The widening gap between modeled fills and actual fills. Execution decay usually shows up before signal decay — a widening gap is often an early crowding signal.
- **Soft breach**: Rolling Sharpe below the backtest 5th percentile, OR drawdown past 1.0× the backtest maximum. Response: cut position size in half.
- **Hard breach**: Drawdown past 1.5× the backtest maximum, OR time underwater past 1.5× the longest backtest stretch. Response: pause the strategy entirely, diagnose.
- **Overfitting finally exposed**: Some strategies never had an edge — the backtest found a pattern in noise and live trading is the first honest out-of-sample test. These die fast, usually within weeks. Recognizable by their behavior: immediate live underperformance from day one.

## The Three Forces That Kill Strategies

Each demands a different response:

| Force | Mechanism | Response |
|-------|-----------|---------|
| **Crowding** | A signal pays because few people trade it. As capital finds it, trades execute earlier at worse prices, return compresses. Most of the 58% post-publication decay is crowding. | **Retire.** No parameter set resurrects a crowded signal. |
| **Regime change** | Every strategy encodes an assumption (momentum assumes trends persist, mean reversion assumes ranges hold). When the regime shifts, the assumption breaks. | **Re-tune.** Lookbacks, volatility filters, sizing rules can sometimes adapt. |
| **Overfitting exposed** | The backtest found a pattern in noise. Live trading is the first honest out-of-sample test. | **Retire fast.** Dies within weeks. Let it. |

## The Four Weekly Monitoring Numbers

Track each against the backtest baseline:

```
1. Rolling Sharpe (90-day)
   Flag: live Sharpe below backtest 5th percentile
   Why: earliest statistical signal of edge erosion

2. Drawdown depth AND duration
   Flag: depth > 1.0× backtest max (soft), > 1.5× (hard)
         duration > 1.0× longest backtest stretch (soft), > 1.5× (hard)
   Why: duration catches the "dying slowly" failure mode a depth check misses

3. Trade-level drift
   Track: hit rate, profit factor, avg win vs avg loss
   Flag: winners getting slightly smaller for 6+ consecutive months
   Why: decay shows up at the trade level before the portfolio level

4. Slippage drift
   Track: gap between modeled fills and actual fills over time
   Flag: widening gap trend
   Why: execution decay precedes signal decay — crowding shows here first
```

## The Kill Criteria (Write Before Deployment)

```
SOFT BREACH (cut size 50%):
  → Rolling Sharpe below backtest 5th percentile
  OR
  → Drawdown past 1.0× backtest maximum drawdown

HARD BREACH (pause strategy, diagnose):
  → Drawdown past 1.5× backtest maximum drawdown
  OR
  → Time underwater past 1.5× longest backtest stretch

After hard breach — diagnosis decides:
  → Crowding diagnosis → RETIRE, reallocate capital
  → Regime shift diagnosis → RE-TUNE with fresh out-of-sample validation
```

"The exact multipliers matter less than the fact that they exist in writing before launch."

## Re-Tune Discipline

After a hard breach with regime change diagnosis:
1. Modify parameters (lookbacks, volatility filters, sizing rules)
2. Run the modified strategy on **data the new parameters have never seen**
3. Compare to original backtest — does the improvement hold?
4. Only redeploy if the fresh out-of-sample run confirms the edit

**Why this matters:** Refitting to the last drawdown produces a strategy that would have avoided exactly the pain you just felt, and nothing else. It is curve fitting in production.

## The Four Ways Traders Get This Wrong

1. **Killing too early**: Abandoning a system inside a statistically normal drawdown. Hopping to the next strategy and paying the switch costs. A Sharpe 1.0 strategy posts a losing year 1 in 6 times — losing months prove nothing.

2. **Killing too late**: Every breach limit was hit 8 months ago, but it came back twice. Sunk cost wearing a quant costume.

3. **Refitting after every losing streak**: Curve fitting in production. Each refit makes the backtest prettier in hindsight; each refit makes the strategy more fragile going forward.

4. **No written rules**: Every decision gets made mid-drawdown, under stress, by the least rational version of yourself.

## The Medallion Calibration Point

By Renaissance's own team's account, Medallion — the best track record ever recorded — was right on roughly 50.75% of its trades. The most profitable system in history lost almost half the time. A losing month proves nothing. The question is whether live performance sits inside the distribution the backtest implied.

## Pre-Deployment Checklist

Record before deploying any strategy:
- [ ] Maximum backtest drawdown
- [ ] Longest backtest underwater period
- [ ] Rolling Sharpe distribution (5th, 25th, 50th, 75th, 95th percentiles)
- [ ] Expectancy (avg win × hit rate − avg loss × miss rate)
- [ ] Modeled slippage per trade
- [ ] Written kill criteria (soft breach threshold, hard breach threshold)

## Weekly Monitoring Checklist

- [ ] 90-day rolling Sharpe vs. backtest percentile (flag if below 5th)
- [ ] Current drawdown depth vs. backtest maximum (flag at 1.0×, pause at 1.5×)
- [ ] Current underwater duration vs. backtest maximum (flag at 1.0×, pause at 1.5×)
- [ ] Hit rate and profit factor vs. baseline (watch for 6-month declining trend)
- [ ] Real fills vs. modeled fills (watch for widening gap trend)

## Questions & Gaps
- The 58% post-publication edge decay figure — is this uniform across strategy types, or do some categories (mean reversion vs. momentum vs. arbitrage) decay faster than others?
- Performance cone construction via resampling assumes the historical trade distribution is stable. How does this break down when the strategy has very few historical trades (e.g., early Polymarket strategies with limited history)?
- The "crowding vs. regime change" diagnosis after hard breach — what diagnostic tests distinguish them in practice? Slippage drift suggests crowding, but is there a more systematic test?
- For prediction market strategies specifically: how do time-horizon differences (5-minute vs. 15-minute vs. multi-day contracts) affect the appropriate monitoring cadence? Weekly monitoring may be too slow for short-duration markets.
- The article doesn't address position sizing recovery after a soft breach size reduction. When and on what criteria do you scale back up?

## Related Notes
- [Linear Regression Alpha Signals](https://github.com/LutherCalvinRiggs/research/blob/main/finance/research/linear-regression-alpha-signals-quant-framework.md) — the Information Coefficient stability test and out-of-sample validation from that note are the signal-building equivalent of the performance cone here. Both are frameworks for distinguishing real edge from noise — that note covers detection before deployment, this note covers monitoring after.
- [Polymarket 6 Trading Bot Types](https://github.com/LutherCalvinRiggs/research/blob/main/finance/research/polymarket-6-trading-bot-types.md) — each of the six bot types is susceptible to the forces described here. Near-resolution bots face regime change risk (resolution mechanics evolve); pure arbitrage bots face crowding risk as more capital discovers the same mispricings.
- [Polymarket Arbitrage Math — $40M](https://github.com/LutherCalvinRiggs/research/blob/main/finance/research/polymarket-arbitrage-math-40-million.md) — the paper documented $39.7M extracted over one year. The framework here explains why that edge will decay: once the strategies are documented and capital flows in, slippage drift appears first, then signal compression. The monitoring stack here is how to know when it's happening.
