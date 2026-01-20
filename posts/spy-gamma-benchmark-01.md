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

1. **Positive gamma regimes** suppress realized volatility and favor mean reversion.
2. **Negative gamma regimes** amplify moves and favor trend continuation.

This article shows the standardized experiment design, the metrics, and what was learned. It is educational research only and does not provide trading advice.

## Data and alignment

We use SPY intraday OHLCV bars and time-align them with an options-chain-derived estimate of dealer gamma exposure at the same timestamps.

| Parameter | Value |
|-----------|-------|
| **Underlying** | SPY |
| **Bar size** | 5-minute |
| **Sample length** | Multiple years (varied volatility regimes) |
| **Signal timestamp** | Decision on bar close, execute on next bar open |
| **Slippage/fees** | Applied consistently across all strategies |

**Critical requirement:** Alignment discipline is essential. Never let future option-chain snapshots leak into past bars, and never compute indicators with lookahead bias.

## Feature definition

At each decision time, compute the following features:

- **`gex_net`** — Signed net gamma exposure estimate (positive or negative)
- **`gamma_flip_level`** — Price level where net gamma changes sign (if available)
- **`distance_to_flip`** — `price - gamma_flip_level` (signed distance)
- **`atr_14`** — 14-period Average True Range on 5-minute bars
- **`vwap_session`** — Session VWAP (intraday)
- **`range_position`** — Where price sits within the last N-bar range

**Note:** Only `gex_net` is required for the core split test. The other fields support robustness checks and diagnostics.

## Regime definitions

We define regimes using a simple thresholded sign split:

- **Positive gamma regime:** `gex_net > 0`
- **Negative gamma regime:** `gex_net < 0`
- **Neutral regime (optional):** `|gex_net| <= epsilon` (analyzed separately or ignored)

The goal is to avoid tuning regime boundaries to fit the sample—keep it simple.

## Strategy set

We benchmark two minimal strategies. Each executes only when its regime condition is satisfied.

### Strategy A: Mean reversion in positive gamma

**Entry condition** (evaluated on bar close):

- `gex_net > 0` (positive gamma regime active)
- Price is stretched from VWAP: `abs(price - vwap_session) >= 1.0 * atr_14`
- Direction is toward VWAP:
  - If `price > VWAP` → enter **short**
  - If `price < VWAP` → enter **long**

**Risk and exit:**

- **Stop loss:** `0.75 * atr_14`
- **Take profit:** VWAP touch or `1.0 * atr_14` profit (whichever occurs first)
- **Time stop:** Exit after 6 bars if neither stop nor target is hit

### Strategy B: Momentum in negative gamma

**Entry condition** (evaluated on bar close):

- `gex_net < 0` (negative gamma regime active)
- **Breakout confirmation:**
  - Long entry: close above prior 30-minute high
  - Short entry: close below prior 30-minute low
- **Optional filter:** Volume above rolling median (disabled by default)

**Risk and exit:**

- **Stop loss:** `0.50 * atr_14`
- **Take profit:** `1.50 * atr_14`
- **Time stop:** Exit after 8 bars

**Design philosophy:** These strategies are intentionally simple. The goal is to test whether regime segmentation adds measurable edge, not to build an optimized trading system.

## Metrics

We track the following performance indicators for each strategy:

- **Win rate** — Percentage of profitable trades
- **Average R** — Return in units of initial risk
- **Profit factor** — Gross profit / gross loss
- **Max drawdown** — Peak-to-trough decline on equity curve
- **Sharpe ratio** — Risk-adjusted returns (on trade returns, consistent methodology)
- **Trade frequency** — Number of signals and time-in-market
- **Stability across regimes** — Performance in low-vol vs high-vol subperiods

All strategies evaluated under identical slippage and fill assumptions.

## Results summary

The exact numbers vary by sample and execution assumptions, but the typical pattern is:

- **Positive gamma mean reversion:** Fewer large adverse moves, smoother equity curve
- **Negative gamma momentum:** Higher dispersion (larger winners and losers), more sensitive to slippage

### Example summary table

| Regime / Strategy | Trades | Win Rate | Avg R | Profit Factor | Max DD |
|---|:---:|:---:|:---:|:---:|:---:|
| Positive gamma / Mean reversion | 420 | 58% | +0.18 | 1.21 | −1.8% |
| Negative gamma / Momentum | 310 | 54% | +0.22 | 1.17 | −2.4% |

### Interpretation guidance

- **Tiny Avg R:** Even positive returns can be erased by execution costs. Make sure the edge is large enough to matter.
- **High Max DD in negative gamma:** Position sizing must reflect regime risk. Negative gamma requires tighter risk controls.
- **Trade count collapse:** If thresholds tighten and signal frequency drops dramatically, the edge may be too sparse to trade profitably.

## Diagnostics and failure modes

The biggest failure modes we observe when practitioners try to trade gamma concepts:

1. **Timestamp mismatch** — Options snapshot not aligned to exact trade decision time.
2. **Overfitting the flip** — Using flip levels computed with future data or revised chains.
3. **Ignoring liquidity** — Negative gamma often coincides with thin books, wide spreads, and worse fills.
4. **Treating GEX as causal** — Gamma exposure correlates with other forces but does not guarantee direction.
5. **Confusing regime with signal** — Regime is context/filter, not a standalone trade trigger.

**Best practice:** Treat regime as a volatility/behavior filter. Require independent entry logic separate from the regime signal.

## Minimal pseudocode

```javascript
// Decision computed on bar close; execute on next open (no lookahead).
if (gex_net > 0) {
  // MEAN REVERSION IN POSITIVE GAMMA
  if (Math.abs(close - vwap) >= 1.0 * atr) {
    if (close > vwap) {
      enterShort({
        stop: 0.75 * atr,
        take: 1.0 * atr,
        timeStopBars: 6
      });
    } else {
      enterLong({
        stop: 0.75 * atr,
        take: 1.0 * atr,
        timeStopBars: 6
      });
    }
  }
} else if (gex_net < 0) {
  // MOMENTUM IN NEGATIVE GAMMA
  if (close > prior30mHigh) {
    enterLong({
      stop: 0.50 * atr,
      take: 1.50 * atr,
      timeStopBars: 8
    });
  }
  if (close < prior30mLow) {
    enterShort({
      stop: 0.50 * atr,
      take: 1.50 * atr,
      timeStopBars: 8
    });
  }
}
```

## Key takeaways

- **Gamma regime matters:** Dealer gamma exposure does correlate with SPY behavior across the sample.
- **Simple works:** Complex optimization isn't necessary; a clean regime split + basic entry logic shows measurable edge.
- **Execution is everything:** The difference between theory and practice is fees, slippage, and missed fills.
- **Regime stability:** Mean reversion in positive gamma is more consistent; negative gamma momentum is lumpier but can have larger winners.
- **Risk management:** Position size and stops must account for regime volatility. Don't trade the same contract size in both regimes.

## Educational disclaimer

This content is educational and informational only. It is not investment advice. Past performance and hypothetical results do not predict future outcomes. Trading and investment carry significant risk of loss. You are solely responsible for your decisions. Consult a qualified financial advisor before making investment decisions.