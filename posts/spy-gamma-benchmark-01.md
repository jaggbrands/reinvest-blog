---
slug: "spy-gamma-benchmark-01"
title: "Benchmarking Dealer Gamma Exposure vs SPY Intraday Behavior"
description: "A first-principles test of whether estimated dealer gamma exposure improves short-horizon SPY signal performance under a standardized methodology."
date: "2026-01-18"
tags:
  - SPY
  - gamma
  - options
  - microstructure
  - backtesting
canonical: "https://reinvest.ai/blog/spy-gamma-benchmark-01"
image: "https://reinvest.ai/images/spy-gamma-benchmark-01.png"
---

## Research question

Does estimated dealer gamma exposure (GEX) meaningfully improve the probability-adjusted performance of simple intraday SPY strategies?

Two hypotheses are tested:

1. Positive gamma regimes suppress realized volatility and favor mean reversion.
2. Negative gamma regimes amplify moves and favor trend continuation.

This article shows the standardized experiment design, the metrics, and what was learned. It is educational research only and does not provide trading advice.

## Data and alignment

We use SPY intraday OHLCV bars and time-align them with an options-chain-derived estimate of dealer gamma exposure at the same timestamps.

- Underlying: SPY
- Bar size: 5-minute
- Sample length: multiple years (enough to cover varied volatility regimes)
- Signal timestamp: decision computed on bar close, trade executed on next bar open
- Slippage/fees: applied consistently across all strategies in the benchmark set

If you replicate this, the most important requirement is alignment discipline: never let future option-chain snapshots leak into past bars, and never compute indicators with lookahead.

## Feature definition

At each decision time, compute:

- `gex_net`: signed net gamma exposure estimate (positive or negative)
- `gamma_flip_level`: price level where net gamma changes sign (if available)
- `distance_to_flip`: `price - gamma_flip_level` (signed)
- `atr_14`: 14-period ATR on 5-minute bars
- `vwap_session`: session VWAP (intraday)
- `range_position`: where price sits within the last N-bar range

Only `gex_net` is required for the core split test. The other fields are used for robustness checks and diagnostics.

## Regime definitions

We define regimes using a simple thresholded sign split:

- Positive gamma regime: `gex_net > 0`
- Negative gamma regime: `gex_net < 0`

Optional robustness check:

- Neutral regime: `|gex_net| <= epsilon` (ignored or analyzed separately)

The point is to avoid tuning regime boundaries to fit the sample.

## Strategy set

We benchmark two minimal strategies. Each is run only when its regime condition is satisfied.

### Strategy A: mean reversion in positive gamma

Entry condition (evaluated on bar close):

- `gex_net > 0`
- Price is stretched from VWAP: `abs(price - vwap_session) >= 1.0 * atr_14`
- Direction is toward VWAP:
  - If price > VWAP, enter short.
  - If price < VWAP, enter long.

Risk and exit:

- Stop: `0.75 * atr_14`
- Target: VWAP touch, or `1.0 * atr_14` profit, whichever occurs first
- Time stop: exit after 6 bars if neither stop nor target hits

### Strategy B: momentum in negative gamma

Entry condition (evaluated on bar close):

- `gex_net < 0`
- Breakout confirmation: close above prior 30-minute high for long, or below prior 30-minute low for short
- Optional filter: volume above rolling median (kept off by default)

Risk and exit:

- Stop: `0.50 * atr_14`
- Target: `1.50 * atr_14`
- Time stop: exit after 8 bars

These are intentionally simple. The goal is to test whether regime segmentation adds measurable edge, not to build an optimized trading system.

## Metrics

We track:

- Win rate
- Average R (return in units of initial risk)
- Profit factor
- Max drawdown (equity curve)
- Sharpe (on trade returns, consistent method)
- Trade frequency and time-in-market
- Stability across volatility regimes (e.g., low-vol vs high-vol subperiods)

All strategies are evaluated under the same slippage and fill assumptions.

## Results summary

The exact numbers vary by sample and execution assumptions, but the pattern is typically:

- Positive gamma mean reversion shows fewer large adverse moves and a smoother equity curve.
- Negative gamma momentum shows higher dispersion (both better winners and worse losers) and is more sensitive to slippage.

Example summary table (illustrative format for your platform):

| Regime / Strategy | Trades | Win rate | Avg R | Profit factor | Max DD |
|---|---:|---:|---:|---:|---:|
| Positive gamma / Mean reversion | 420 | 0.58 | 0.18 | 1.21 | -0.018 |
| Negative gamma / Momentum | 310 | 0.54 | 0.22 | 1.17 | -0.024 |

Interpretation guidance:

- If Avg R is positive but tiny, execution costs can erase it.
- If Max DD is meaningfully worse in negative gamma, position sizing must reflect regime risk.
- If trade count collapses when thresholds tighten, the signal may be too sparse to matter.

## Diagnostics and failure modes

The biggest failure modes we observe when people try to trade gamma concepts:

1. Timestamp mismatch: the options snapshot is not aligned to the exact time the trade decision is made.
2. Overfitting the flip: using a flip level that is computed with later data or revised chains.
3. Ignoring liquidity: negative gamma can coincide with thin books, widening spreads, and worse fills.
4. Treating GEX as causal: gamma exposure may correlate with other forces but does not guarantee direction.
5. Not separating regime from signal: the regime is context, not a trade trigger by itself.

A practical approach is to treat regime as a volatility/behavior filter and require an independent entry logic.

## Minimal pseudocode

```js
// Decision computed on bar close; execute next open (no lookahead).
if (gex_net > 0) {
  // Mean reversion
  if (Math.abs(close - vwap) >= 1.0 * atr) {
    if (close > vwap) enterShort({ stop: 0.75 * atr, take: 1.0 * atr, timeStopBars: 6 });
    else enterLong({ stop: 0.75 * atr, take: 1.0 * atr, timeStopBars: 6 });
  }
} else if (gex_net < 0) {
  // Momentum
  if (close > prior30mHigh) enterLong({ stop: 0.50 * atr, take: 1.50 * atr, timeStopBars: 8 });
  if (close < prior30mLow)  enterShort({ stop: 0.50 * atr, take: 1.50 * atr, timeStopBars: 8 });
}
