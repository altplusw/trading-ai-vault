---
tags: [order-flow, features]
---

# 07 - Order Flow

## Purpose

Compute order-flow **features** from raw market data. Features are inputs to a strategy,
not conclusions. This layer is deterministic and testable.

## The governing rule

> **Treat these as FEATURES, not truths.**

Never hard-code statements like *"price will hunt the liquidation cluster"* or
*"price up + CVD down = reversal."* These are **hypotheses to be measured**, not laws.

For every feature X and every narrative claim about it, the project must be able to answer:

```
Under condition Y, does feature X have measurable predictive value for outcome Z,
out of sample, after costs?
```

If that cannot be demonstrated, the feature stays a research note in [[25 - Research Log]]
and never gates a trade.

### Worked example — CVD divergence

- **Narrative:** "rising price with falling CVD implies reversal."
- **Testable form:** condition = up-move > X%; feature = ΔCVD < 0; outcome = price falls >Y%
  within Z bars.
- **Required:** measured base rate, sample size, out-of-sample confirmation, cost-adjusted.

Until that table exists, CVD divergence is a *feature in a feature set*, not a signal.

## Feature catalogue

Each entry states what it measures. **Availability depends on the provider**
([[06 - Market Data]]) — unsupported features return `None`, never `0`.

### Microstructure

| Feature | Definition | Notes |
|---|---|---|
| `spread_bps` | (ask − bid) / mid × 10⁴ | execution cost proxy; risk gate |
| `book_imbalance` | (Σ bid size − Σ ask size) / (Σ bid + Σ ask) | within depth band; band must be stated |
| `depth_bid` / `depth_ask` | summed size within N bps of mid | N must be fixed and recorded |
| `top_of_book_pressure` | size at best bid/ask only | more fragile than banded imbalance |
| `microprice` | weighted mid by adjacent size | short-horizon fair value |

### Aggressor flow

| Feature | Definition | Notes |
|---|---|---|
| `buy_volume` / `sell_volume` | volume by taker side over window | requires trade-side data |
| `volume_delta` | buy − sell | provider-dependent definition; pin it |
| `cvd` | cumulative volume delta since session/window start | **anchor must be explicit** |
| `trade_imbalance` | buy trades / (buy + sell trades) | differs from volume-based |
| `large_trade_ratio` | share of volume in top-decile prints | liquidity/toxicity proxy |
| `trade_rate` | prints per second | activity regime |

### Futures / positioning — **optional, venue-dependent**

| Feature | Availability | Notes |
|---|---|---|
| `open_interest` | usually paid or partial free | treat OI change ± price as context, never a signal |
| `oi_change_pct` | derived | |
| `funding_rate` | usually free on perps | extremes indicate crowded positioning |
| `liquidations_long` / `_short` | usually paid/aggregated | forced-flow context |
| `liquidation_clusters` | **usually estimated, not ground truth** | ⚠️ see below |

### ⚠️ Liquidation clusters are estimates

"Where are the liquidations?" is almost always **modelled from OI distribution**, not
observed directly. Any provider's cluster levels are an estimate with its own assumptions.

Therefore:

- Treat cluster levels as **one input among many**, with the provider and method recorded.
- Never claim a cluster level is *the* liquidation level.
- Record `liquidation_source` and `liquidation_method` with every stored snapshot.
- If the method cannot be identified, do not use the feature.

## Definitions must be pinned

Ambiguity here silently corrupts backtests. The project must document, for every feature:

- Window length (e.g. CVD over what period?).
- Depth band for book features.
- Session anchor for cumulative measures.
- How the provider defines taker side (if it doesn't, don't use it).
- Handling of gaps, missing trades, out-of-order prints.

Store these definitions in the DB with each feature set so a backtest can be replayed
identically.

## FeatureSet

```python
@dataclass(frozen=True)
class FeatureSet:
    symbol: str
    as_of: float            # source timestamp — the ONLY time that matters
    computed_at: float      # local wall clock, for latency measurement
    features: Mapping[str, float | None]   # None = unsupported/unavailable
    definitions_version: str
```

Frozen. `as_of` is what backtests compare against the decision timestamp. `computed_at` is
diagnostics only. `None` is meaningful and must propagate — never coerce to `0.0`.

## Dependency

Consumes `MarketDataProvider` ([[06 - Market Data]]). Emits into the
`MarketStateEngine` ([[03 - System Architecture]]).

**The LLM never sees a raw order book.** Book data is large and structured; the Main AI
receives the **computed** `FeatureSet`, not the L2 ladder. This is both a token-budget
decision and a correctness one.

## Laya input constraint

Laya is a **text** encoder ([[05 - Laya]]) with a small token budget (512 tokens on the
English checkpoint). It cannot consume a 50-level ladder.

Therefore Laya receives a **compact derived representation** — the state engine's summary,
not the raw book. This is a hard design constraint, and one of the main reasons the state
engine must be deterministic and terse.

## Testing

- Each feature has a unit test with a hand-built input and expected output.
- Definition-pinning tests (window, band, anchor).
- `None` propagation for unsupported features.
- Timestamp correctness: `as_of` ≤ decision timestamp, always.
- Gap/out-of-order handling.

---

[[06 - Market Data]] · [[05 - Laya]] · [[08 - Strategy Engine]] · [[13 - Backtesting]]

## See also

- [[25 - Research Log]] — feature-vs-outcome measurement tables
- [[14 - Calibration]] — how feature-derived probabilities are validated
- [[27 - Known Limitations]] — historical order-book replay