---
tags: [calibration, validation]
---

# 14 - Calibration

## Purpose

Measure whether Laya's probabilities mean what they appear to mean **on our data, for our
questions**. Do not assume they transfer.

## Why this exists

Two verified facts make this mandatory:

1. **Laya's shipped checkpoints are near chance on domain tasks** — 0.362 vs a 0.461
   majority baseline on the project's own benchmark ([[05 - Laya]]).
2. **They fail confidently.** The English checkpoint scores 0.000 on Khmer while reporting
   95.2% confidence.

A probability is only useful if it means something. Until measured, treat every Laya
probability as an uncalibrated score — useful for *ranking*, not for *sizing*.

## Which number to measure

Laya returns two different confidences. They must not be confused:

| Field | Definition | Calibrated by temperature scaling? |
|---|---|---|
| `answer_confidence` | `max(p)` — probability of the reported answer | **Yes** |
| `confidence` | `1 − H(p)/log(k)` — concentration | **No** |

**Calibrate and gate on `answer_confidence` only.** `confidence` shifts with option count
and is not a probability of being right. Laya's own documentation recommends exactly this
distinction.

## Stored calibration records

Every Laya evaluation persists one row:

| Field | Purpose |
|---|---|
| `timestamp` | |
| `symbol`, `timeframe`, `market` | slice |
| `question_id` | per-question calibration |
| `model_version`, `question_battery_version` | so versions are comparable |
| `predicted_label`, `probability` | the prediction |
| `answer_confidence`, `confidence` | both, for comparison |
| `outcome` | what actually happened, with its own `as_of` |
| `horizon` | the prediction window — **must be recorded** |

**Record the horizon.** "P(bullish) = 0.7" is meaningless without "over the next 15 minutes."
An unfalsifiable calibration study is worthless.

## Metrics

| Metric | Purpose |
|---|---|
| **Brier score** | mean squared error of probabilities; lower better |
| **Reliability curve** | predicted vs observed, binned |
| **ECE** | expected calibration error |
| **MCE** | maximum calibration error — worst bin |
| **Brier skill score** | vs a baseline (always-predict-frequency) |
| Per-question breakdown | a good average can hide one badly wrong question |
| Per-regime breakdown | calibration often differs by market state |
| Per-bucket breakdown | few options behave differently from many |

Baseline matters: a trivially well-calibrated model that always predicts the base rate has
a *good* Brier score and *zero* skill. Always report the baseline alongside.

## Interpretation rules

- Report **sample size** with every calibration figure. Below ~100 outcomes per bucket,
  say so explicitly.
- A confidence value that never appears in your data makes that range uncalibrated —
  report it as unmeasured, not as good.
- Aggregate ECE hides per-question failure. Always check the breakdown.
- Test on **out-of-sample** data. Calibration fitted on the evaluation set is meaningless.

## Recalibration

If (and only if) measurement shows miscalibration, fit a deterministic, versioned transform.

Laya supports temperature scaling, fitted per `(question type, option count)` bucket. Its
own tooling clamps temperatures to `[0.5, 5.0]` and **falls back to 1.0** with a warning for
invalid values.

⚠️ **A fallback prevents a load failure; it does not make a probability calibrated.** A
clamped-to-1.0 bucket is *uncalibrated*, and must be recorded as such.

Recalibration requirements:

1. Deterministic — same data ⇒ same transform.
2. Versioned and recorded alongside every decision.
3. Fitted on held-out data only.
4. Original and calibrated values **both stored**, so post-hoc comparison is possible.
5. Never fitted on the evaluation set.

## Gates using confidence

`risk.min_confidence` defaults to **0.0** deliberately ([[09 - Risk Engine]]). It should be
raised only when:
1. Calibration has been measured on our data.
2. The Brier skill score is meaningfully positive.
3. Enough samples exist in the relevant regime.
4. The threshold has been validated out-of-sample.

Raising it earlier rejects arbitrarily — protecting nothing.

## Testing

- Records are written for every Laya call.
- Brier/ECE against hand-computed fixtures.
- Baseline comparison present in reports.
- Horizon is mandatory (missing ⇒ record rejected).
- Recalibration is deterministic and versioned.
- Clamped/fallback temperatures flagged as uncalibrated in reporting.

---

[[05 - Laya]] · [[09 - Risk Engine]] · [[13 - Backtesting]] · [[16 - Self Improvement]]

## See also

- [[25 - Research Log]] — calibration results
- [[21 - Testing]] — calibration metric tests
- [[27 - Known Limitations]] — the near-chance baseline problem