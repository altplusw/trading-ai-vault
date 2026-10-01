---
tags: [backtesting, research, integrity]
---

# 13 - Backtesting

## Purpose

Provide **evidence**, honestly. A backtest is a hypothesis test with assumptions made
explicit. It is never proof of future profitability.

## The governing rule

> A successful backtest is **not** proof. It is one more piece of evidence, with a known
> set of weaknesses, that must survive out-of-sample and paper validation.

Every report states: data range, symbol, timeframe, assumptions, fees, slippage model,
fill model, sample size, and whether it is in-sample.

## Integrity — the most important section of this document

### Look-ahead bias

**Only information available before the decision timestamp may be used.**

| Leak | Cause | Guard |
|---|---|---|
| Future candle | using the in-progress bar | closed candles only |
| Future indicator | computing an indicator over data past `as_of` | assert `as_of ≤ decision_ts` for every input |
| Order-book lookahead | using a snapshot after the trade | replay with monotonic timestamps |
| Trade leakage | attributing a later print to an earlier decision | strict `<` comparison, never `≤` |
| Survivorship | only symbols that still exist | universe fixed as of the period start |
| Train/eval contamination | shuffling before splitting | **temporal split only** |
| Parameter over-optimisation | tuning on the test set | untouched final holdout |
| Selection bias | reporting only good strategies | all runs recorded, including failures |

### Mandatory safeguards

1. **Temporal splits only.** Never shuffle. Train → validation → out-of-sample, in time order.
2. **Assertion at the boundary.** Every feature carries `as_of`. The engine raises if any
   input's `as_of` exceeds the decision timestamp. Not a warning — an exception.
3. **Purged gaps** between train and validation to prevent label bleed across the boundary.
4. **Walk-forward** as the primary validation method, not a single split.
5. **Costs always on.** Fees and slippage are never zero in a reported result.
6. **Final holdout touched once.** If it has been viewed, it is no longer out-of-sample;
   say so and get a new one.

### Conservative biases (deliberate, documented)

Where the truth is unknowable from the data, assume the worse:

- Intrabar ambiguity (both stop and target hit) ⇒ **assume the stop filled first**.
- Missing volume ⇒ assume the fill was worse.
- Unknown fill queue position ⇒ assume you were not filled on favourable orders.

These biases make backtests pessimistic, which is the correct direction for a tool whose
purpose is to prevent over-trading.

## Simulation model

Reuses the paper broker ([[10 - Paper Trading]]) — **one execution implementation**, not two.
If the backtester and the paper engine disagree, results are not trustworthy.

Simulated: fees, slippage, partial fills, position sizing, risk limits, entries/exits,
stops, targets, latency assumptions where relevant.

### Data availability constraints — documented limits

| Data | Historical availability | Consequence |
|---|---|---|
| OHLCV | good | full simulation |
| Public trades (with taker side) | partial | CVD/delta backtests limited to available range |
| Historical L2 order book | **rare; usually normalised and downsampled** | microstructure backtests are approximate |
| Liquidations | usually paid, usually short history | limited window |
| Open interest | partial free, good paid | context only |
| Funding | generally good | usable |

**Never simulate order-book replay we do not have.** If historical depth is unavailable,
the backtest is candle-based and must say so. Presenting candle-based results as
order-flow results is a form of dishonesty.

## Metrics

Reported together, never selectively:

total return · net P&L · gross P&L · win rate · loss rate · trade count · average trade ·
average win · average loss · largest win · largest loss · fees · total slippage ·
max drawdown · Sharpe · Sortino (where meaningful) · exposure · profit factor · expectancy ·
average hold duration · equity curve.

### Reporting rules

- **Always** show trade count alongside win rate.
- Profit factor undefined at zero gross loss ⇒ report as N/A, not infinity.
- Sharpe on < ~30 trades is noise ⇒ report with that caveat or omit.
- Show the **distribution**, not only the mean.
- State the assumptions inline, not in a footnote.

## Dataset splits

| Split | Purpose | Rule |
|---|---|---|
| **Train** | fitting parameters | may be optimised |
| **Validation** | model selection | may be compared across candidates |
| **Out-of-sample** | single honest estimate | **touched once** |
| **Walk-forward** | primary robustness check | rolling, multiple folds |

Do **not** define one arbitrary metric as a pass/fail gate and optimise to it. Gates exist
to catch nonsense, not to be the optimisation target ([[26 - Decisions]]).

## Run records

Every backtest run is persisted with: strategy version, data range, symbol, timeframe,
assumptions, all parameters, all metrics, equity curve, code version, timestamp, dataset
checksum. **Failed and unpromising runs are kept** — a research log containing only
successes is a selection bias ([[25 - Research Log]]).

## Testing

- Look-ahead assertion fires on a deliberately poisoned dataset.
- Train/validation split preserves time order (assert monotonic timestamps).
- Intrabar stop-before-target policy.
- Fees/slippage applied and verifiable in the ledger.
- Purge gap exists between splits.
- Determinism: same inputs ⇒ identical equity curve.
- Metrics against hand-computed fixtures.
- A negative-result backtest produces an equal-quality report.

---

[[10 - Paper Trading]] · [[14 - Calibration]] · [[16 - Self Improvement]] · [[07 - Order Flow]]

## See also

- [[06 - Market Data]] — what history is actually available
- [[21 - Testing]] — backtest integrity tests
- [[26 - Decisions]] — why gates are not optimisation targets
- [[27 - Known Limitations]] — unavailable historical data