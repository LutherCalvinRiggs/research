# Linear Regression Alpha Signals: A Quant Framework

**Source:** https://x.com/rohonchain/status/2063992615721435154
**Saved:** 2026-06-10
**Tags:** finance, research, fundamentals, economics, trading, tools

---

## TL;DR
Linear regression is the foundational tool of institutional quantitative finance — not a stepping stone to something more exotic. The key insight: alpha is the intercept that survives after every known risk factor is stripped out, and the job is proving it's real (not noise) through out-of-sample testing, Information Coefficient stability, Newey-West correction, and multiple testing discipline. Even real signals decay; the durable edge is the research process that replaces them faster than they erode.

## Key Concepts & Terms

- **Alpha (Jensen's alpha)**: The return a strategy earns that cannot be explained by exposure to known risk factors. Not raw profit — the intercept that remains after stripping out beta. If you earned 20% but the market also earned 20%, your alpha is zero. Michael Jensen formalized this in 1968; it's still how institutional performance is judged.
- **Beta**: Sensitivity to a known risk factor. The return you collect for taking market risk. If you have beta without alpha, you're riding a wave, not demonstrating skill.
- **OLS (Ordinary Least Squares)**: The method that finds the best-fit line by minimizing the sum of squared gaps between the line and the data. Big misses are penalized far more than small ones (squared). The institutional workhorse for signal building.
- **`sm.add_constant()`**: The single most important line in any OLS model for trading. Creates the intercept term (alpha). Omitting it forces the line through zero and throws away the most important number in the output.
- **R-squared**: Fraction of return variation explained by the model. Counterintuitively, a high R-squared on return prediction is a warning, not a trophy. Real signals are weak — R² of 0.01–0.02 can be wildly profitable if persistent. R² of 0.9 almost certainly means look-ahead bias.
- **p-value**: Answers: "If this factor truly had no effect, how likely is it I'd see a relationship this strong by chance?" p = 0.62 means 62% chance you're looking at noise. Conventional bar: below 0.05. Institutional bar for new signals: below 0.01 (after multiple testing correction).
- **t-statistic**: Coefficient ÷ standard error. Rule of thumb: above 2 (absolute value) ≈ the 0.05 threshold. Serious researchers demand ≥3 on a new signal because they know how many signals they tested before this one.
- **Information Coefficient (IC)**: The correlation between a factor's predicted value and the actual forward return, computed repeatedly across time. A mean IC above ~0.03–0.05 that's stable across many periods beats a single gorgeous backtest. Stability across regimes is the real signal of signal quality.
- **Newey-West standard errors**: Corrects OLS standard errors for autocorrelation and heteroskedasticity in financial data (returns and volatility cluster in time). Raw p-values will lie — usually flattering the signal. Using Newey-West is the mark of someone who knows what they're doing.
- **Multiple testing problem**: If you test 100 useless signals at p < 0.05, about 5 will pass by luck alone. The most common reason backtests lie. Defense: Bonferroni correction — divide your threshold by the number of signals tested. Tested 50? Demand p < 0.001.
- **Factor crowding**: When a signal becomes widely known and traded, the trading itself pushes out the inefficiency. Simple momentum signals have been shown to decay along a predictable curve as capital piles in, accelerating sharply once factor investing became cheap via ETFs.
- **Signal decay**: Every real alpha signal eventually dies. The edge lives in market inefficiency; once it's traded, the inefficiency is arbitraged away. Durable edge = research speed, not any single signal.

## The Core Model

```
rₜ = α + β Xₜ + εₜ

where:
  rₜ  = asset return at time t
  α   = alpha — the intercept — your candidate edge
  β   = sensitivity to the factor Xₜ
  Xₜ  = your predictive factor at time t
  εₜ  = noise — the part no factor explains
```

The entire discipline reduces to one question: is α statistically different from zero after all known factors are accounted for?

## Building a Signal: The Code

### Single-Factor Model
```python
import numpy as np
import statsmodels.api as sm

# y = asset returns you want to predict
# X = your factor series (prior return, momentum, value metric, etc.)

X = sm.add_constant(X)   # CRITICAL: creates the alpha intercept term
model = sm.OLS(y, X).fit()
print(model.summary())

# Pull the numbers that matter
print("Alpha:", round(model.params['const'], 5))
print("Alpha p-value:", round(model.pvalues['const'], 4))
print("t-stats:\n", model.tvalues)
print("R-squared:", round(model.rsquared, 4))
```

### Multi-Factor Model (Strip Out Known Risk)
```python
import statsmodels.api as sm

# factors = DataFrame with columns: market_return, size, value, momentum
# y = your strategy returns

X = sm.add_constant(factors)
model = sm.OLS(y, X).fit()

alpha   = model.params['const']    # residual edge after all known factors
p_alpha = model.pvalues['const']   # probability this is noise

print("Alpha:", round(alpha, 5))
print("Alpha p-value:", round(p_alpha, 4))
# Remaining alpha = the slice your edge that no known factor explains
```

### Newey-West Corrected Significance (Institutional Default)
```python
# Correct for autocorrelation and volatility clustering
model = sm.OLS(y, X).fit(cov_type='HAC', cov_kwds={'maxlags': 5})
print(model.summary())
# Raw p-values lie in financial data. This is non-negotiable.
```

### Out-of-Sample Validation (Non-Negotiable)
```python
split = int(len(y) * 0.8)
X_train, X_test = X[:split], X[split:]
y_train, y_test = y[:split], y[split:]

model = sm.OLS(y_train, X_train).fit()
oos_pred = model.predict(X_test)
# If alpha disappears on unseen data, it was never real.
```

## Reading the Output: The Professional Frame

| Number | What beginners read | What researchers read |
|--------|--------------------|-----------------------|
| Return size | "How big is it?" | Irrelevant until significance is confirmed |
| p-value | Ignore or celebrate | Primary question: could this be luck? |
| R-squared | Higher = better | High R² on returns = warning sign (look-ahead bias?) |
| t-statistic | Ignore | Must be ≥2; demand ≥3 on new signals |
| Sign of coefficient | Ignore | Does this make economic sense? If not, it's a fluke |

**The shift**: The beginner asks "how big is the return?" The professional asks "how confident am I this return isn't an accident?" That single change in the question is the entire difference.

## The Five-Test Gauntlet

A real signal must survive all five. Almost nothing does, and that's the point:

1. **Out-of-sample test** — Build on 80%, test on 20% the model never saw. If alpha collapses, it was never real.
2. **IC stability** — Compute IC (correlation of factor value vs. actual forward return) repeatedly across time. Mean IC > 0.03–0.05, stable across regimes = genuine. Single gorgeous backtest = suspect.
3. **Newey-West significance** — Re-run with HAC standard errors. If significance vanishes, the raw p-values were lying to you.
4. **Multiple testing correction** — Bonferroni: divide p-value threshold by number of signals tested. Tested 50 signals? Demand p < 0.001, not 0.05.
5. **Economic logic** — If you can't explain why the sign makes sense, you found a coincidence.

## The Renaissance Lesson

Jim Simons' Medallion Fund: ~66% gross annual returns 1988–2018. The best record ever recorded.

- Right on only **50.75%** of trades
- Across millions of trades, that razor-thin edge compounded into >$100B
- **Not predicting the future** — pulling a tiny, real, repeatable signal out of an ocean of noise
- Never stopped researching for a single day in 30 years

The replacement loop is the whole career. One signal is a trade. A research process that produces real signals faster than they decay is a franchise.

## The Closing Question Worth Sitting With

> "If real alpha decays the moment it becomes widely known, then publishing a signal destroys it, while hoarding a signal means it still slowly dies as others independently discover it. So where does durable edge actually come from — the signals themselves, or the speed of the research process that replaces them?"

This is the question that separates traders chasing one big win from quants building a machine that prints edge for decades. Renaissance's answer: the machine. Not Medallion's positions. The research infrastructure.

## Questions & Gaps
- The article covers OLS thoroughly but doesn't discuss when linear regression breaks down — non-linear relationships, regime changes, fat-tailed distributions. What are the practical limits for prediction market applications specifically?
- IC threshold of 0.03–0.05 — is this universal across asset classes, or does it differ for prediction markets where resolution mechanics create unusual return distributions?
- The Bonferroni correction is deliberately brutal. Are there less conservative multiple testing corrections (Benjamini-Hochberg, for example) that practitioners actually prefer?
- Factor crowding decay curves are mentioned but not quantified — over what time horizon do simple momentum signals typically decay in prediction markets vs. equity markets?
- The article doesn't address transaction costs or slippage — for Polymarket specifically, how does the CLOB spread affect the minimum IC threshold needed for a strategy to be profitable after execution costs?

## Related Notes
- [Polymarket 6 Trading Bot Types](https://github.com/LutherCalvinRiggs/research/blob/main/finance/research/polymarket-6-trading-bot-types.md) — the Repricing/Fair Value Model Bot described there is exactly the OLS framework applied in practice: build a fair value model, compare to market price, buy the undervalued side. This article gives the mathematical foundation for that bot type.
- [Polymarket Arbitrage Math — $40M](https://github.com/LutherCalvinRiggs/research/blob/main/finance/research/polymarket-arbitrage-math-40-million.md) — the Bregman projection framework there is the non-linear generalization of the OLS signal framework described here. Both are about finding the gap between current market price and a calculated fair value.
