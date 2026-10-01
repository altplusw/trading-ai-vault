---
tags: [limitations, open-questions, critical]
---

# 27 - Known Limitations

## Purpose

An honest register of what does **not** work, what is unverified, and what is undecided.
Read this before trusting any component or result.

Nothing here is a bug to be quietly fixed. Some are permanent properties of the chosen
tools. Some need user decisions.

---

## L-01 · Laya's shipped checkpoints are near chance on domain tasks ⚠️ CRITICAL

**Verified.** On the Laya project's own typed-decisions benchmark the shipped English and
multilingual checkpoints score **0.362** and **0.352**, against a **0.461 majority-class
baseline** and 0.318 random. The 0.766 headline comes from a fine-tuned checkpoint (~2 GPU
hours).

**Impact.** Laya output is **not** a trading signal out of the box. Fine-tuning on labelled
market data is required (Phase 10). Until then, treat Laya as a cost filter.

**Cannot be worked around.** Only by fine-tuning and calibration measurement.

---

## L-02 · Laya fails confidently on out-of-domain input ⚠️

**Verified.** The English checkpoint scores **0.000 accuracy at 95.2% confidence** on Khmer.
Macro ECE across 51 languages is 0.733 raw.

**Impact.** Confidence gating cannot protect against a model that is confidently wrong.
Confidence is therefore not a safety mechanism.

---

## L-03 · Laya cannot read images ⚠️

**Verified.** `serialize_state()` is `str` or `json.dumps`. No PIL, cv2, or image path exists.

**Impact.** **Any chart-image or screenshot analysis is impossible with Laya.** Requires a
separate vision model, which is out of scope for now. This kills the original "read the chart"
idea — correctly.

---

## L-04 · Laya degrades badly above ~20 options

**Verified.** Options share one token budget. Project-measured: **Banking77 scores 0.425
across 77 options** versus a competitor's 0.870.

**Impact.** Keep questions ≤ ~20 options. Use `predict_shortlist` or question splitting
beyond that.

---

## L-05 · Laya's token budget is small

**Verified.** 512 tokens (English) / 1024 (multilingual, up to 8192 with RoPE). Real state
room is `max_len − head_len − 1`, **not** `max_len − head_max_len`.

**Impact.** Feature sets must be compact and terse. Truncation must be logged and checked.

---

## L-06 · "OpenMarket" could not be verified as a public API ⚠️ UNRESOLVED

**Finding.** Repeated searching returned CoinAPI, Amberdata, CoinMetrics, CoinGlass,
CoinGecko, CoinMarketCap, ccxt, Massive, Tiingo — but **no OpenMarket**. No OpenMarket
endpoints, pricing, rate limits, or coverage are documented in this vault, because none were
verified.

**Impact.** Cannot plan against it. **No provider chosen.** Recommendation: start on Binance
public endpoints ([[06 - Market Data]]).

**Resolution needed:** the user may have a specific OpenMarket product in mind. If so, its
documentation must be supplied or its base URL identified before Phase 3.

---

## L-07 · Market data provider undecided

**Impact.** Blocks Phase 3. Symbol format, timeframe semantics, rate limits, and cost all
depend on it.

---

## L-08 · Historical order-book replay rarely available

**Finding.** Providers that offer historical L2 typically supply **normalised, downsampled
snapshots** that do not bit-match live data (stated by CoinAPI). Full tick-level historical
depth is generally paid.

**Impact.** Microstructure backtests are approximate. Candle-based backtests must be labelled
as such. **Do not present candle results as order-flow results.**

---

## L-09 · Historical liquidation data is short and often paid

**Impact.** Any liquidation-based hypothesis has a small sample. Likely insufficient for
statistical significance. Feature may be dropped rather than half-used.

---

## L-10 · Liquidation "clusters" are estimates

**Finding.** Cluster levels are typically modelled from OI distribution, not observed.

**Impact.** Must be recorded with source and method; never presented as ground truth.

---

## L-11 · Security gaps in the Phase 1 foundation

| Gap | Status |
|---|---|
| Log **redaction** filter | ⬜ not implemented — secrets could reach logs |
| HTTP API key auth | ⬜ not implemented |
| Request/concurrency limits on the HTTP layer | ⬜ not implemented |
| Audit events | ⬜ not implemented |

**Impact.** Acceptable for a localhost-only dev build. **Must be closed before any
non-localhost exposure** (Phase 2).

---

## L-12 · Autonomous mode is a flag, not a loop

**Impact.** `start_autonomous_mode` records intent only. No monitoring loop exists (Phase 11).
No trade is possible in this build.

---

## L-13 · No market data, no paper execution, no risk engine, no backtester

**Impact.** The system cannot trade, cannot be backtested, and cannot produce P&L. Phase 1 is
a foundation, not a product.

---

## L-14 · Order types limited to market and limit

No stop/iceberg/TWAP, no partial-fill realism beyond the documented model, no queue-position
modelling.

---

## L-15 · Latency benchmarks not yet run

**Verified as estimates only.** Laya CPU latency (~193–464 ms per the Laya project) and
end-to-end tool-call latency are **unmeasured on this hardware**.

**Impact.** D010 (Laya on CPU) is a reasoned hypothesis, not a measurement. Phase 5 must
benchmark.

---

## L-16 · Reference project `jev-trader` differs substantially

**Finding.** It is a TypeScript/Bun market-making bot on Monad (Kuru MON-USDC), one AI
decision per ~300 ms block, posting post-only limit orders one tick inside the touch. 12
commits. It shares our *architecture principle* but not our problem.

**Impact.** Useful for the adapter pattern (`Model` interface with `JevModel`/`MockModel`)
and its `dryRun` default. **Not** a template for this system. Its strategy must not be assumed
profitable — it is a live market-making bot on one venue, and no independent profitability
evidence was found.

---

## L-17 · Laya project maturity and provenance

**Finding.** The Laya repository's first commit is 2026-09-18, three days after Jev's public
release (2026-09-15). It uses the same "System One" framing and the same "RLCD" acronym, and
ships a checkpoint file named `rl_agent_config.json`. Its author states the work predates Jev;
public git history cannot corroborate that.

**Impact.** **Not** a technical objection — the code is Apache-2.0 and was read directly. But
it means: pin the version (D017), and do not build critical assumptions on long-term API
stability.

---

## Open questions for the user

| # | Question | Blocks |
|---|---|---|
| Q1 | Which market data provider? Is "OpenMarket" a specific product? | Phase 3 |
| Q2 | Main LLM: local model, or a free remote provider? | Phase 2 |
| Q3 | Spot or perpetual futures? | Phase 3 |
| Q4 | Symbols and timeframe to start with? | Phase 3 |
| Q5 | Budget for paid historical data, if any? | Phase 9 |
| Q6 | Which decision battery questions actually matter? | Phase 5 |

---

[[00 - Home]] · [[05 - Laya]] · [[06 - Market Data]] · [[26 - Decisions]]

## See also

- [[25 - Research Log]] — what has been measured so far
- [[13 - Backtesting]] — data-availability constraints
- [[30 - Coding Agent Instructions]] — rule 4 covers updating this file