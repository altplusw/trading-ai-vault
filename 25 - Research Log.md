---
tags: [research, log, evidence]
---

# 25 - Research Log

## Purpose

A dated record of **what was measured, what was found, and what remains unknown.** Keeping
failed and null results is as important as keeping successes — a log containing only wins is
a selection bias and will mislead the research loop ([[16 - Self Improvement]]).

## Format

```
### YYYY-MM-DD — Title
**Question:** what we wanted to know
**Method:** how we tested it
**Result:** what we found, with numbers and sample sizes
**Conclusion:** what we believe now
**Status:** confirmed / refuted / inconclusive
**Next:** what to do about it
```

Rules:

- **Always report sample size.** A 70% win rate over 6 trades is not a result.
- **Record refuted hypotheses** with the same care as confirmed ones.
- **Record variant counts** when parameter searches were run (multiple-comparisons guard).
- Never edit a past entry. Add a correction underneath.
- No profitability claims. Report measured metrics with assumptions.

---

## 2026-10-02 — Vault created; Laya API verified from source

**Question.** What does Laya actually do, and what is its real interface?

**Method.** Read the source at commit `6d942c9` (v0.3.22), cloned to `../laya-repo`:
`common.py` (model + sequence builder), `agent.py` (runtime), `router.py` (routing),
`lang.py` (detection), `serve.py` (HTTP), plus the shipped checkpoint config.

**Result.**

- Prompt-conditioned discriminative classifier (PET-style) on ModernBERT. One forward pass,
  no generation. Sequence: `[CLS] <type> question: <ins> [SEP] [MASK] opt0 … [SEP] <state> [SEP]`.
- HTTP: `POST /v1/systemone`, `POST /v1/systemone/batch` (≤64 states), `GET /health`.
  Forwarded controls: `model, max_len, head_max_len, task, lang, lang_guess, min_confidence`.
  Refused (422): the five hook arguments.
- Three question types. `score` returns a **probability-weighted expectation**, not argmax.
- `answer_confidence` = max probability (calibrated by temperature scaling);
  `confidence` = 1 − normalised entropy (not calibrated).
- Context 512 (English) / 1024, extendable to 8192 via RoPE. `head_max_len` 192/256.
- Server limits: 64 questions, 100 choice options, 32 score levels, 512 total options,
  2 MiB body, 8192 token budget cap, 16 concurrent.
- Checkpoints: 421M ModernBERT-large (English), 322M mmBERT-base (multilingual).
- Shipped typed-decisions accuracy 0.362 / 0.352 vs **0.461 majority baseline**. Khmer 0.000 at
  95.2% confidence on the English checkpoint.
- No image/vision path. No training code in the repo (notebooks only).

**Conclusion.** Technically sound and fast. **Not** a zero-shot trading signal. Integration
must treat it as a filter, encode state as compact text, argmax `score` distributions, and
gate on `answer_confidence`.

**Status.** Confirmed.

**Next.** Benchmark CPU latency on this hardware (Phase 5). Re-verify the API before
integrating — upstream had moved 174 commits.

---

## 2026-10-02 — "OpenMarket" not found

**Question.** Is OpenMarket a usable market-data source?

**Method.** Multiple targeted web searches for its API, documentation, and pricing.

**Result.** No OpenMarket product appeared. Results were CoinAPI, Amberdata, CoinMetrics,
CoinGlass, CoinGecko, CoinMarketCap, ccxt, Massive, Tiingo, Bitquery.

**Conclusion.** **Cannot be confirmed to exist as a public API.** No OpenMarket behaviour
documented in this vault. Not fabricating endpoints.

**Status.** Inconclusive — needs clarification.

**Next.** Ask the user whether they mean a specific product and request its documentation or
base URL. Interim recommendation: Binance public endpoints.

---

## 2026-10-02 — Market data free tiers surveyed

**Question.** What crypto data is available free of charge?

**Method.** Reviewed provider documentation and pricing pages.

**Result.** Binance public REST/WS is keyless and covers price, book, trades, funding, OI, and
liquidation streams — the strongest free option. CoinAPI has a 10 req/s / 1000 req/day free
tier but explicitly serves **normalised, downsampled** historical L2 that does not match live
data. CoinGlass and Amberdata require payment for the liquidation/OI depth this project wants.
ccxt is a free portability library.

**Conclusion.** The entire system is buildable at **$0** on Binance public data. Historical
order-book replay is the one feature likely to require payment, and it can be deferred with
candle-based backtests that are honestly labelled.

**Status.** Confirmed.

**Next.** Confirm the provider (Q1) before Phase 3.

---

## 2026-10-02 — Reference project `jev-trader` reviewed

**Question.** What can be learned from the named architectural reference?

**Method.** Read its README and `.env.example`.

**Result.** TypeScript/Bun market-making bot on Monad, Kuru MON-USDC. One AI decision per
~300 ms block; posts a post-only limit order one tick inside the touch, replacing the last.
Uses a `Model` interface with `JevModel` and `MockModel`; `MODEL=mock|jev`. Defaults to
`DRY_RUN=true` and requires `PRIVATE_KEY` for live. Tracks `late` (missed the block ⇒ hold) —
a fail-closed pattern. 12 commits, 2.7k stars.

**Conclusion.** The **adapter pattern** and **dry-run-by-default** are worth adopting. The
problem (sub-second market making on one L2) is **not** this project's problem. No independent
evidence of profitability was found.

**Status.** Confirmed as an architectural reference only.

**Next.** Mirror the `Model` interface shape and the dry-run default.

---

## 2026-10-02 — VRAM budget for concurrent local models

**Question.** Can the Main AI and Laya share the RTX 3060 Ti?

**Method.** Arithmetic from parameter counts and common quantisations.

**Result.** Laya English bf16 + activations ≈ 1.2 GB. An 8B Q4 model + KV cache ≈ 6–6.5 GB.
Total ≈ 7.8 GB against 8 GB — no headroom, plus CUDA context duplication across processes.

**Conclusion.** Do not co-locate. Run **Laya on CPU**, GPU for the Main AI.

**Status.** Confirmed by arithmetic; **not yet measured** (L-15).

**Next.** Benchmark Laya on CPU in Phase 5. If insufficient, reduce Main AI model size rather
than assuming.

---

## Templates

### Feature-vs-outcome study

```
### YYYY-MM-DD — Does <feature> predict <outcome>?
**Feature:** definition, window, band, anchor, provider
**Condition:** the filter applied
**Outcome:** definition, horizon
**Data:** range, symbol, sample size, split
**Result:** base rate vs conditional rate; deltas; confidence interval
**Costs applied:** fees, slippage
**Status:** confirmed / refuted / inconclusive
```

### Model calibration study

```
### YYYY-MM-DD — Calibration of Laya <question> on <market>
**Question set:** battery version
**Model version:** Laya version + our fine-tune, if any
**Horizon:** (required — a probability without one is meaningless)
**Sample:** N, periods, regimes covered
**Brier / BSS / ECE / MCE:** with baseline
**Breakdown:** per question, per regime, per option-count bucket
**Verdict:** usable for gating / ranking only / not usable
```

### Strategy comparison

```
### YYYY-MM-DD — <strategy> v<n> out-of-sample
**Hypothesis:** what should happen, stated before the test
**Data / split:** ranges, walk-forward folds
**Variants tried:** N (record this — it matters)
**Result:** all metrics from [[13 - Backtesting]], with trade count
**Verdict:** promoted / rejected, with reason
```