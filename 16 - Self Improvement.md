---
tags: [research-loop, governance]
---

# 16 - Self Improvement

## Purpose

A **research loop** that analyses outcomes and proposes changes — where proposals are
validated, reviewed, and applied by a human. Never automatically.

## The governing rule

> The AI may **propose**. A human **applies**. Nothing self-modifies in production.

Uncontrolled self-modification by a model optimising against its own recent outcomes is the
most likely way this system produces a confidently wrong result. Guard against it
explicitly, not politely.

## Pipeline

```
TRADING DATA (paper only)
   ↓
PERFORMANCE ANALYSIS
   ↓
ERROR ANALYSIS
   ↓
AI PROPOSES CHANGE        ← proposal only
   ↓
CANDIDATE created (status=draft)
   ↓
BACKTEST  →  VALIDATION  →  OUT-OF-SAMPLE
   ↓
REVIEW by human
   ↓
APPROVE → deploy to paper  |  REJECT → archive with reason
```

Every step is recorded. A rejection is a first-class outcome, stored with its reason.

## What may be proposed

| Category | Example | Validation needed |
|---|---|---|
| New strategy | momentum breakout | full backtest + OOS |
| Changed filter | require Laya regime = trending | full backtest + OOS |
| New Laya question | add order-flow question | backtest + **calibration study** |
| Removed Laya question | drop a question with no lift | backtest + calibration |
| Changed parameter | stop distance 1.5×ATR → 2×ATR | walk-forward, not single split |
| New data feature | add `large_trade_ratio` | backtest + feature study |
| Bug fix | off-by-one in CVD window | unit test, no backtest needed |
| Risk rule adjustment | raise max position 20% → 25% | ⚠️ **human decision only** |

**Risk-rule changes are never AI-initiated.** The model may *note* that a rule appears
binding; a human decides. Risk limits are policy, not optimisation targets.

## Analysis inputs

From paper and backtest runs only: trade outcomes, missed opportunities, false signals,
Laya decisions vs outcomes, **risk rejections** (an underrated signal — a rule that
constantly blocks may be miscalibrated), execution quality vs modelled, performance by
regime, per-strategy results.

### Error taxonomy

Classify before proposing fixes. Without this, analysis degenerates into narrative.

| Class | Meaning |
|---|---|
| `entry_error` | entered where the thesis was wrong |
| `exit_error` | right thesis, wrong exit |
| `sizing_error` | correct direction, size disproportionate to edge |
| `filter_error` | a filter blocked a good trade or allowed a bad one |
| `execution_error` | modelled fill differed materially from simulated |
| `data_error` | missing/stale/wrong data drove the decision |
| `calibration_error` | confidence was over/under-stated |
| `luck` | within expected variance — **not** a pattern |

`luck` matters. Most individual losses are noise. A loop that treats every loss as
evidence produces strategies fitted to noise.

## Overfitting guards

- **Never** deploy on in-sample improvement alone.
- Multiple-comparison awareness: testing 50 variants and keeping the best guarantees a
  falsely good result. Record the number of variants tried, always.
- Minimum sample size and minimum holding period per strategy.
- Walk-forward as the standard check.
- Prefer **fewer, simpler** changes. Every added parameter costs degrees of freedom.
- Require the improvement to exceed noise, not merely to be positive.

## Nightly research mode

Optional scheduled analysis. It reviews trades, missed opportunities, false signals, Laya
decisions, risk rejections, execution quality, regimes and per-strategy performance, then
**writes proposals to the research log** for human review.

It does not deploy, does not trade, and does not modify production state. Its only write
target is the proposals table.

## Provenance

Every candidate records: parent version, proposer (model id + prompt version), timestamp,
the data range used, all parameters, and the full validation results. A candidate without
provenance is not reviewable and is discarded.

## Testing

- Proposals never modify production state (integration test asserting no writes outside the
  research tables).
- Candidates default to `status=draft`.
- Promotion requires recorded evidence of every prior gate.
- Approval is impossible without a human action token.
- Risk-rule proposals require human approval even when everything else passes.
- Overfitting guard: a candidate beating the holdout only by noise is rejected.
- Variant count is recorded.

---

[[13 - Backtesting]] · [[14 - Calibration]] · [[08 - Strategy Engine]] · [[25 - Research Log]]

## See also

- [[26 - Decisions]] — why nothing self-modifies
- [[20 - Security]] — what the research loop may never touch
- [[28 - Future Ideas]] — ideas stay here until they pass the pipeline