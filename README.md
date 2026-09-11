# Statistical Arbitrage in Cryptocurrencies

A research project exploring statistical arbitrage strategies in crypto markets, built end-to-end in Python: data acquisition, signal research, rigorous statistical testing, and out-of-sample backtesting with realistic transaction costs.

## Summary

This project tests two families of statistical arbitrage strategies on a universe of 400+ crypto assets (Binance, hourly/daily data, 2021–2026):

1. **Cross-sectional reversal** (with a volume-conditioning refinement) — found a real, statistically broad signal that **did not survive realistic transaction costs**.
2. **Cointegration-based pairs trading** — the final strategy, achieving a **0.58 net Sharpe ratio out-of-sample** with **near-zero correlation to BTC (0.01)**.

The negative result on the reversal side is reported deliberately: a large part of this project is about rigorously testing whether a promising-looking signal survives realistic frictions and proper out-of-sample validation, not just reporting whichever backtest looks best.

## Methodology

### 1. Data
- 400+ USDT pairs pulled from Binance, hourly granularity, history extended back to each coin's earliest available date (as far back as 2017 for major coins).
- Universe filtered by liquidity (24h volume) and manually screened to exclude stablecoins, wrapped/staked assets, and tokenized equity products that don't represent independent crypto price dynamics.

### 2. Reversal strategy
- Cross-sectional momentum/reversal signal tested across multiple horizons (1h–1YE) using pooled correlation and per-coin sign-agreement.
- Found a strong, broad reversal effect at short horizons (up to ~85% of coins agreeing on direction).
- Refined with a volume-conditioning filter (reversal following low-volume moves vs. high-volume moves), which sharpened the raw signal considerably.
- **Result:** even the volume-conditioned version did not survive realistic transaction costs (20bps) at any tested rebalancing frequency, due to persistently high turnover relative to signal strength.

### 3. Pairs trading (final strategy)
- **Pair selection:** correlation pre-screen (>0.7 on daily log returns) → Engle-Granger cointegration test on **log prices** (critical — raw-price regressions produce unstable hedge ratios for assets on very different price scales) → **Bonferroni and Benjamini-Hochberg correction** for multiple testing across all candidate pairs.
- **Strict in-sample/out-of-sample split:** pair selection uses only in-sample data; entry/exit thresholds are tuned only on in-sample performance; the out-of-sample period is touched exactly once, for final evaluation.
- **Trading logic:** rolling (90-day) beta and alpha (not a single static regression) to build a time-varying spread; z-score entry/exit rules; a stop-loss on extreme spread moves.
- **Portfolio construction:** pairs weighted by statistical confidence tier (Bonferroni survivors > Benjamini-Hochberg survivors > naive-significant pairs), correctly timed so a day's weight matches the position that generated that day's return.
- **Transaction costs:** modeled at the level of each underlying coin leg, accounting for turnover from both position changes and the continuously-drifting hedge ratio — not just pair-level allocation changes.

## Results (out-of-sample)

| Metric | Value |
|---|---|
| Sharpe Ratio (net of costs) | 0.58 |
| Correlation with BTC | 0.01 |
| Max Drawdown | ~-4% |
| Blended Sharpe (90% strategy / 10% BTC) | 0.93 |

The near-zero correlation to BTC indicates the strategy's returns come from relative price movements between paired assets, not from directional crypto market exposure — a genuinely diversifying return stream rather than disguised beta.

## Key methodological lessons (and mistakes caught along the way)

- **Raw-price regressions can produce nonsensical hedge ratios** for coins on very different price scales (e.g. a $0.50 coin vs. a $0.000015 coin) — cointegration testing and spread construction should use log prices.
- **Universe growth over the backtest window is a real source of survivorship-bias-like inflation** if not controlled for — an expanding coin universe changes what "top-N by return" means over time.
- **Pair selection must be validated strictly out-of-sample.** An earlier version of this project used a correlation pre-screen computed on the full dataset before applying an in-sample/out-of-sample split for cointegration testing — this leaked information from the test period into pair selection and was corrected.
- **Position weights must be timed to match the position that generated the return**, not the current day's signal — a subtle off-by-one-day bug can silently drop real return contributions on every exit day.
- **A raw Sharpe ratio isn't the whole story.** Reporting the strategy's Sharpe alongside its correlation to a simple benchmark (BTC buy-and-hold) is what actually demonstrates diversification value.

## Limitations

- Backtest period (~5.5 years) is short relative to typical professional standards (5–20 years); results should be interpreted with appropriately wide confidence intervals.
- The 90/10 BTC blend that improves Sharpe was identified by testing several allocations on the out-of-sample period; it is reported as a supplementary diversification result, not the headline strategy metric.
- The alpha estimate (vs. BTC) is positive but not statistically significant at this sample size (t-stat ≈ 1.0) — a longer track record would be needed to confirm it with confidence.
- The portfolio currently trades all naive-significant pairs at reduced weight alongside the statistically strictest survivors; a cleaner ablation (Bonferroni/BH-only portfolio vs. full set) is a natural next step.

## Tech stack

Python · pandas · numpy · statsmodels (OLS, Engle-Granger cointegration, ADF, multiple-testing correction) · python-binance · matplotlib

## Data

Raw price data is not included in this repository (too large for version control). The notebook re-pulls data directly from the Binance API via `python-binance`.
