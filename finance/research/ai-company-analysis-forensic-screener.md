# How to Read Any Company With AI Like an Analyst

**Source:** https://x.com/gemchange_ltd/status/2060757358297469365
**Saved:** 2026-06-10
**Tags:** finance, research, tools, fundamentals, economics, technology

---

## TL;DR
A five-layer framework for AI-assisted forensic financial analysis of any public company or crypto token. L0: free structured data via SEC EDGAR + DefiLlama. L1: four forensic math scores that flag manipulation, distress, earnings quality, and financial strength. L2: AI year-over-year diff of Risk Factors and MD&A sections to surface what management quietly changed. L3: crypto-specific supply schedule and holder concentration screening. L4: orchestrated pipeline combining all layers. Includes a full working Python script.

## Key Concepts & Terms
- **Beneish M-Score**: Eight-input forensic formula that detects earnings manipulation probability. Above -2.22 = investigate. The heaviest input is total accruals over total assets — the fastest way to fake earnings is booking income that never showed up as cash. Enron printed -1.89 in 1998 when a Cornell MBA class ran it; Wall Street still had it rated buy.
- **Altman Z-Score**: Bankruptcy risk composite mixing profitability, leverage, and asset efficiency. Below 1.81 = distress zone. Above 2.99 = safe zone.
- **Sloan Accruals Ratio**: Earnings quality signal. Earnings made of cash are real; earnings made of accruals reverse. `(net_income - cash_from_operations) / total_assets`. Drift past ±25% → earnings are likely an accounting mirage about to unwind.
- **Piotroski F-Score**: 9 yes/no questions on whether a company is getting financially stronger. ≥6 = healthy. Covers profitability, leverage, liquidity, operating efficiency — each is either trending right or wrong.
- **EDGAR (SEC)**: Free US government API serving every public company's filings. Two killer features: (1) full-text search across all filings (e.g., search "material weakness" across the entire market to get a short watchlist in one second), (2) structured financials — every line item, every quarter, machine-readable, going back years.
- **Risk Factors diff**: Year-over-year comparison of a company's Risk Factors section in its 10-K. New language = something changed that lawyers felt they had to disclose. Deleted language = a risk the company no longer wants to advertise. Neither shows up in press releases.
- **MD&A (Management Discussion & Analysis)**: Management's narrative explanation of the year. Combined with the Risk Factors diff, catches the stories management buries in footnotes — where Enron's actual fraud lived.
- **TVL (Total Value Locked)**: Crypto protocol equivalent of assets under management — total value of assets deposited. Via DefiLlama free API.
- **Token unlock schedule**: The vesting calendar for team and VC allocations. When they unlock, holders who got in near zero can sell into the market. Red flag: any single unlock over 5% of circulating supply. Three shapes: cliff (violent single-day dump), linear vest (slow daily bleed), emissions (activity-based). Cliff into a VC wallet ends portfolios.
- **Holder concentration**: What percentage of supply is held by a small number of wallets. If team/VC wallets hold majority supply, retail is the exit liquidity by design.
- **edgartools**: The correct Python library for EDGAR. Parses 10-Ks, 8-Ks, insider Form 4s, 13F fund holdings into clean Python objects. Ships an MCP server so Claude can fetch real filings rather than hallucinate numbers. `pip install edgartools`, no API key required.
- **FinanceToolkit**: Open-source repo with 150+ ratios including all four forensic scores, with formulas written in the open so you can audit any number you don't trust. Pair with FMP key for data.

## The Five-Layer Stack

### L0: Data Sources

| Source | Type | Cost | Best for |
|--------|------|------|---------|
| `edgartools` | Python library | Free | EDGAR filings — structured, MCP-ready |
| SEC EDGAR full-text search | API | Free | Find every company that mentioned "material weakness" this quarter |
| BamSEC | Web reader | Free (mostly) | Clean filing reader, side-by-side compare |
| DefiLlama | API | Free | Crypto TVL, fees, revenue, unlock schedules |
| Token Terminal | Platform | ~$350/mo | Standardized crypto P/E equivalents across chains |

```bash
pip install edgartools anthropic
export SEC_IDENTITY="Your Name your@email.com"  # SEC requires this header
```

### L1: Forensic Math Scores

Run all four simultaneously across your watchlist. Only read the names that flag.

| Score | What it detects | Flag threshold |
|-------|----------------|----------------|
| Beneish M-Score | Earnings manipulation | Above -2.22 |
| Altman Z-Score | Bankruptcy risk | Below 1.81 (distress), above 2.99 (safe) |
| Sloan Accruals | Earnings quality | \|accruals/assets\| > 25% |
| Piotroski F-Score | Financial strengthening | Below 6/9 |

```python
# Beneish M-Score (eight inputs → one number)
def beneish_m_score(t, p):
    DSRI = d(d(t.receivables, t.sales), d(p.receivables, p.sales))        # receivables growth vs sales
    GMI  = d((p.sales - p.cogs)/p.sales, (t.sales - t.cogs)/t.sales)     # margin deterioration
    AQI  = d(1 - d(t.current_assets + t.ppe_net, t.total_assets),
             1 - d(p.current_assets + p.ppe_net, p.total_assets))         # asset quality
    SGI  = d(t.sales, p.sales)                                             # sales growth
    DEPI = d(d(p.depreciation, p.depreciation + p.ppe_net),
             d(t.depreciation, t.depreciation + t.ppe_net))                # depreciation rate
    SGAI = d(d(t.sga, t.sales), d(p.sga, p.sales))                       # SG&A inflation
    TATA = d(t.net_income - t.cfo, t.total_assets)                        # total accruals (heaviest weight)
    LVGI = d(d(t.total_liabilities, t.total_assets),
             d(p.total_liabilities, p.total_assets))                       # leverage growth
    return (-4.84 + 0.92*DSRI + 0.528*GMI + 0.404*AQI + 0.892*SGI
            + 0.115*DEPI - 0.172*SGAI + 4.679*TATA - 0.327*LVGI)

# Altman Z-Score
def altman_z_score(t):
    wc = t.current_assets - t.current_liabilities
    return (1.2*d(wc, t.total_assets) + 1.4*d(t.retained_earnings, t.total_assets)
            + 3.3*d(t.ebit, t.total_assets) + 0.6*d(t.market_cap, t.total_liabilities)
            + 1.0*d(t.sales, t.total_assets))

# Piotroski F-Score (9 yes/no checks)
def piotroski_f_score(t, p):
    s  = (t.net_income > 0)                                                # profitable
    s += (t.cfo > 0)                                                       # cash flow positive
    s += (d(t.net_income, t.total_assets) > d(p.net_income, p.total_assets))  # ROA improving
    s += (t.cfo > t.net_income)                                            # cash beats accruals
    s += (t.long_term_debt < p.long_term_debt)                             # debt declining
    s += (d(t.current_assets, t.current_liabilities) >
          d(p.current_assets, p.current_liabilities))                      # liquidity improving
    s += (t.shares <= p.shares)                                            # no dilution
    s += (d(t.sales-t.cogs, t.sales) > d(p.sales-p.cogs, p.sales))       # margin expanding
    s += (d(t.sales, t.total_assets) > d(p.sales, p.total_assets))        # asset turnover improving
    return int(s)

# Sloan Accruals
def sloan_accruals(t): return d(t.net_income - t.cfo, t.total_assets)
```

**Important caveats**: Beneish runs on last year's data — manipulation may already be unwinding by the time you see it. It misses some real frauds and false-flags clean ones. A bad score means open the filing, never short on the number alone.

### L2: AI Document Diff

Wrong way: paste the whole 10-K into a chat and ask "is this a good company?" — it drowns and tells you what you want to hear.

Right way: year-over-year diff of specific sections.

```python
# Risk Factors diff prompt
"""
Compare these two Risk Factors sections from consecutive annual filings.
Report ONLY what is NEW this year or what was REMOVED. Quote the new language.
Ignore boilerplate that appears in both.
End with one sentence: does anything here change the risk?

LAST YEAR: {last_year_risk_factors}
THIS YEAR: {this_year_risk_factors}
"""
```

What to look for:
- New paragraph about customer concentration → a big client is wobbling
- Deleted line about a key supplier → a relationship ended
- New going-concern language → auditors are worried
- New related-party risk → founders are extracting value through side deals
- Enron's fraud: the press releases were lies; the footnotes (off-balance-sheet entities) were not

Apply the same diff to MD&A and footnotes. `edgar-crawler` repo rips Risk Factors and MD&A into clean JSON — use it to avoid regexing through HTML.

### L3: Crypto Screening

**The three protocol numbers** (via DefiLlama):
- Fees = everything users pay → gross revenue
- Revenue = the slice the protocol keeps → net revenue
- Earnings = revenue minus tokens printed to bribe users → actual profitability

**Unlock schedule red flags**:
- Any single unlock > 5% of circulating supply → investigate
- Cliff shape into VC wallet → portfolio-ending event
- Real example: Arbitrum's first cliff unlocked ~100% of circulating supply in one day, all to early holders who got in near zero

**Holder concentration tools**:
- Arkham — free for individuals, entity tracing (traced billions in stolen Bitcoin back to a hack)
- Nansen — "smart money" wallet labeling across chains, ~$49/mo
- Dune — 100K+ community SQL dashboards, free tier sufficient for most needs

### L4: Orchestration Pipeline

```
1. Tool layer  — function calling/MCP wrapping EDGAR + FMP + DefiLlama
                 AI fetches real numbers; never invents them (non-negotiable)
2. Screen layer — forensic scores + onchain checks run automatically
3. Read layer   — year-over-year diff on survivors
4. Synthesis    — model writes memo with citation on every claim
                 You read the memo instead of 200 pages
```

**Key repos**:
- `virattt/ai-hedge-fund` — team of AI agents modeled on famous investor philosophies. The investor-persona gimmick is a gimmick. As a free lesson in how to orchestrate analysis agents (chain data-fetcher → screener → reasoner), it's the best teacher on GitHub right now. Do not trade it live.
- `OpenBB-finance/OpenBB` — open-source Bloomberg terminal with MCP server. Powerful but heavy setup. Worth it for one cockpit; overkill for occasional screening.
- `FinanceToolkit` (JerBouma) — 150+ ratios, formulas auditable in the open, pairs with FMP key.

### L5: The Complete Forensic Screener Script

Full working Python script that:
1. Pulls real EDGAR filings via `edgartools`
2. Computes all four forensic scores
3. Optionally runs the year-over-year Risk Factors diff via Claude API
4. Prints a verdict: CLEAN or INVESTIGATE with specific flags

```bash
# Setup
pip install edgartools anthropic
export SEC_IDENTITY="Your Name your@email.com"
export ANTHROPIC_API_KEY="sk-..."   # optional, only for the diff

# Run
python forensic_screener.py AAPL
python forensic_screener.py TSLA NVDA SMCI      # screen several at once
python forensic_screener.py SMCI --diff          # add risk-factor diff
```

Output format:
```
============================================================
  SMCI
============================================================
  Beneish M  : +1.23   (> -1.78 = investigate)
  Altman Z   : 1.52    (< 1.81 distress, > 2.99 safe)
  Piotroski F: 4/9     (>= 6 strong)
  Sloan Accr : +31.2%  (|x| > 25% red flag)

  VERDICT: INVESTIGATE
    - M-Score +1.23 - earnings-manipulation risk
    - Z-Score 1.52 - financial distress zone
    - Accruals +31.2% - earnings-quality red flag
    - F-Score 4/9 - not strengthening
```

The script uses customizable thresholds (the article uses -1.78 for M-Score rather than the classic -2.22 to be more conservative). Tag aliases in the EDGAR data fetcher cover standard filers; exotic ones may need an alias added.

## Paid Tool Landscape (Honest Map)

| Tool | Best for | Cost | Verdict |
|------|---------|------|---------|
| Hudson Labs (ex-Bedrock AI) | Auto red-flag extraction, going-concern language, material weaknesses | ~$100/mo | Best value per dollar if you read filings seriously |
| AlphaSense + Tegus | Institutional default + thousands of former-exec interviews | ~$15–20K/seat | Only if firm pays |
| Daloopa | DCF modeling with every number hyperlinked to its filing | Enterprise | Overkill unless modeling is your job |
| Fintool | AI-first US equities, citations, standing alerts | Mid-tier | Good middle ground |

## Questions & Gaps
- The M-Score threshold is set at -1.78 in the script (more conservative than the academic -2.22) — what's the empirical false positive rate at each cutoff on a large dataset?
- `edgartools` MCP server — how stable is this for Claude integration? Any known issues with the tag alias mapping for unusual filers?
- The crypto earnings metric (fees - tokens printed to incentivize users) is noted but not shown how to calculate programmatically via DefiLlama — is this data directly available or requires manual derivation?
- The Risk Factors diff approach assumes lawyers write truthfully under the threat of lawsuit — how often do companies materially omit risks that later materialize without prior disclosure?
- For prediction market applications specifically: can EDGAR forensic scores be used to build alpha signals on event contracts where the underlying company's financial health is material (e.g., "will Company X file for bankruptcy within 12 months")?

## Related Notes
- [Linear Regression Alpha Signals](https://github.com/LutherCalvinRiggs/research/blob/main/finance/research/linear-regression-alpha-signals-quant-framework.md) — the forensic scores here (Beneish, Altman, Piotroski, Sloan) are multi-factor models: each combines multiple inputs into a single signal. The validation discipline from that note (out-of-sample testing, IC stability, Newey-West) applies equally to these forensic signals.
- [Polymarket Arbitrage Math](https://github.com/LutherCalvinRiggs/research/blob/main/finance/research/polymarket-arbitrage-math-40-million.md) — the AI-assisted dependency detection layer in that paper (DeepSeek-R1 classifying market pairs) uses the same "AI reads structured data, flags anomalies" pattern as the Risk Factors diff described here.
- [Agent Platform Security Checklist](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/agent-platform-security-checklist.md) — "Tool layer first — function calling or MCP servers wrapping EDGAR so the model fetches real numbers and never invents them. Non-negotiable." This is the same principle as the checklist's credential injection proxy and read-only tool separation.
