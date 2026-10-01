---
tags: [roadmap, plan]
---

# 29 - Implementation Roadmap

## Purpose

Ordered phases with explicit **approval gates**. Do not skip a gate. Do not progress to a
phase until the previous gate passes.

## Current position

| | |
|---|---|
| **Phase** | 0 — Documentation |
| **Status** | Vault created; awaiting review |
| **Next gate** | Gate 2 — Architecture reviewed |
| **Existing code** | Phase 1 skeleton exists as a *reference for direction*, not as a passing gate |

---

## Approval gates

| Gate | Condition | Status |
|---|---|---|
| **1** | Documentation complete | ✅ vault created |
| **2** | Architecture reviewed | ⬜ **awaiting user** |
| **3** | Implementation plan reviewed | ⬜ |
| **4** | Foundation tests pass | ⬜ |
| **5** | Market-data tests pass | ⬜ |
| **6** | Paper-trading tests pass | ⬜ |
| **7** | Risk-engine tests pass | ⬜ |
| **8** | Backtesting validation passes | ⬜ |
| **9** | Paper autonomous mode passes | ⬜ |
| **10** | Live-trading integration requires **explicit user approval** | ⬜ not planned |

⚠️ **Gates are not silently skipped.** If a gate is bypassed, it is recorded as bypassed
with a reason in [[25 - Research Log]].

---

## Phase 0 — Documentation ✅

Create and validate the vault. Cross-link. Record decisions, limitations, open questions.

**Exit:** Gate 1.

---

## Phase 1 — Project Foundation

Repository structure · configuration · logging · database · test framework · **provider
interfaces**.

Key deliverables:
- `LLMProvider`, `DecisionEngine`, `MarketDataProvider`, `Broker` protocols defined.
- Config with validation and the paper-only invariant.
- Structured JSON logging **with secret redaction** (closes gap L-11).
- Database schema for all specified tables.
- Test harness; safety gates green.

**Exit:** Gate 4.

---

## Phase 2 — Main AI

Provider abstraction complete · connect to the selected provider · chat loop · structured
tool calling · HTTP auth + request limits.

Also closes the remaining L-11 security gaps before any non-localhost exposure.

**Exit:** part of Gate 4; architecture unchanged.

---

## Phase 3 — Market Data

Market-data adapter (provider per Q1) · candles · live prices · statistics · staleness
enforcement · provider circuit breaker.

**Depends on:** Q1 (provider), Q3 (spot vs perp), Q4 (symbols/timeframe).

**Exit:** Gate 5.

---

## Phase 4 — Order Flow

Order book · depth · spread · CVD · volume delta · and — **only where the provider supports
them** — liquidations, open interest, funding. Feature definitions pinned and versioned.

**Exit:** part of Gate 5.

---

## Phase 5 — Laya

⚠️ **Re-verify the Laya API first** (D017) — it was a 12-day-old project with ~170
commits/week at time of writing.

Deliverables:
- `LayaDecisionEngine` implementing the interface.
- Compact state encoding (D016).
- Question battery per Q6, each with a stated purpose.
- `score` → label conversion by argmax (D015).
- **Benchmark CPU latency on this hardware** (closes L-15, validates D010).
- Structured logging of every evaluation.

**Exit:** part of Gate 4. Benchmarks recorded in [[25 - Research Log]].

---

## Phase 6 — Paper Trading

Account · orders · fills · positions · P&L · fees · slippage · ledger. Deterministic, tested
against hand-computed cases and property-based invariants.

**Exit:** Gate 6.

---

## Phase 7 — Risk Engine

Hard risk boundaries · emergency stop · fail-closed behaviour · **audit events**.

Includes log redaction if not already complete.

**Exit:** Gate 7.

---

## Phase 8 — Trade Journal

Complete per-trade context captured at decision time (D007). "Why did you make that trade?"
answered from stored rows.

**Exit:** part of Gate 7.

---

## Phase 9 — Backtesting

Deterministic historical simulation **reusing the paper broker** (D006). Look-ahead
assertions. Metrics. Walk-forward. Every run recorded, including failures.

**Exit:** Gate 8.

---

## Phase 10 — Calibration

Measure Laya probabilities on our data (D005, D015). Brier/BSS/ECE/MCE with baseline.
Per-question and per-regime breakdowns. Only then consider raising
`RISK_MIN_CONFIDENCE`.

**Exit:** part of Gate 8. Feeds Gate 9.

---

## Phase 11 — Autonomous Paper Trading

Monitoring loop · Laya filtering · escalation budgets · graded autonomy starting at
**level 1 (observe)** · immediate stop · emergency stop · per-dependency fail-closed tests.

**Exit:** Gate 9.

---

## Phase 12 — Dashboard

Complete UI: chart, order book, CVD panel, AI activity timeline, journal browser, settings.
All libraries vendored locally — **no CDN**.

**Exit:** feature complete. Live trading still not authorised.

---

## Phase 13 — Research Loop

AI proposes strategy/filter/question/feature changes. Validated through
backtest → out-of-sample → **human approval**. Nothing auto-deploys. Risk-rule changes are
human-initiated only.

**Exit:** feature complete.

---

## Phase 14 — Optional Live Trading ⛔ NOT PLANNED

**Requires Gate 10: explicit, deliberate, user authorisation.**

Not on the critical path. Not scheduled. If ever pursued:

1. A **new broker implementation** in a separate package behind the existing `Broker`
   interface — not a flag flip (D001, D008).
2. Extensive paper validation first, documented.
3. Separate credentials with **no withdrawal permission**.
4. Explicit human confirmation for every session.
5. Independent security review.

**Nothing in Phases 0–13 depends on this existing.**

---

## Blockers

| Question | Blocks | Owner |
|---|---|---|
| Q1 — market data provider (is "OpenMarket" real?) | Phase 3 | **user** |
| Q2 — Main LLM: local or free remote? | Phase 2 | **user** |
| Q3 — spot or perpetual? | Phase 3 | **user** |
| Q4 — symbols and timeframe? | Phase 3 | **user** |
| Q5 — budget for historical data? | Phase 9 | **user** |
| Q6 — which Laya questions matter? | Phase 5 | research |

---

[[00 - Home]] · [[26 - Decisions]] · [[27 - Known Limitations]] · [[30 - Coding Agent Instructions]]

## See also

- [[02 - Goals and Non Goals]] — scope boundaries
- [[21 - Testing]] — what each gate requires
- [[13 - Backtesting]] — Gate 8 criteria
- [[15 - Autonomous Trading]] — Gate 9 criteria