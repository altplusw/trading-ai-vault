---
tags: [decisions, adr]
---

# 26 - Decisions

Architecture decision log. Format: **Decision → Reason → Consequence → Status.**

New decisions are **appended**. Superseded decisions are marked, never deleted — knowing
what was tried and rejected is part of the value.

---

## D001 — Paper trading only; no real-money code

**Decision.** The initial system is paper-only. There is no exchange SDK installed or
imported and no code path that could place a live order.

**Reason.** Safety and validation. A model-driven trading system must be measurable and
reproducible before real money is at risk.

**Consequence.** `REAL_TRADING` cannot be enabled; startup raises. Live trading, if ever, is
a new broker implementation behind the existing interface, in a separate package, gated by
explicit human authorization.

**Status.** Active. Verified by `tests/test_safety.py`.

---

## D002 — Models advise, code decides

**Decision.** The Main AI and Laya may only **propose**. Deterministic code validates and
executes. No model can widen a limit, bypass a check, or disable the kill switch.

**Reason.** Hard safety boundary. Model output is untrusted input regardless of how good the
model is.

**Consequence.** The risk engine takes limits only from configuration. The broker's entry
point requires a `risk_decision_id`. This is the single most important structural decision
in the project; everything else is subordinate to preserving it.

**Status.** Active.

---

## D003 — Fail closed everywhere

**Decision.** Every dependency failure results in **no trade**. Market data stale, Laya down,
LLM down, database down, risk engine error, malformed model output.

**Reason.** In trading, acting on absent information is worse than not acting.

**Consequence.** Availability is deliberately sacrificed for safety. Errors surface in the UI
rather than being smoothed over.

**Status.** Active.

---

## D004 — Every component replaceable

**Decision.** `LLMProvider`, `DecisionEngine`, `MarketDataProvider`, and `Broker` are
interfaces. The application never imports a provider module outside its adapter.

**Reason.** The user may switch models, data sources, or decision engines without rewriting
the system. Providers change pricing and coverage constantly.

**Consequence.** Enforced by test: a module importing `openai`, `binance`, or `laya` outside
its adapter is a defect.

**Status.** Active.

---

## D005 — Laya is a cost filter first, a signal second

**Decision.** Until Laya is fine-tuned on labelled market data, its role is filtering
uninteresting events before the Main AI runs — **not** gating trades on its output.
`LAYA_MIN_CONFIDENCE` and `RISK_MIN_CONFIDENCE` default to `0.0`.

**Reason.** Verified: the shipped checkpoints score 0.362 against a 0.461 majority-class
baseline on the project's own typed-decisions benchmark, and fail *confidently* (0.000
accuracy at 95.2% confidence on Khmer). A confidence floor against an uncalibrated model
rejects arbitrarily — it looks like a safeguard while providing none.

**Consequence.** An untuned Laya still saves Main AI compute, which is real value. Signal
requires fine-tuning plus calibration measurement ([[14 - Calibration]]).

**Status.** Active. Revisit after Phase 10.

---

## D006 — One execution implementation shared by backtest and paper

**Decision.** The backtester reuses the paper broker rather than having its own execution
logic.

**Reason.** Two execution implementations drift. If they disagree, backtest results do not
describe the system being measured.

**Consequence.** Backtest must support historical price input. Slight added complexity in the
broker interface; a large gain in trustworthiness.

**Status.** Active.

---

## D007 — Persist reasoning at decision time, never reconstruct it

**Decision.** Laya output and model reasoning are stored verbatim in the same transaction as
the decision. Never written or edited afterwards.

**Reason.** A model asked later why it acted produces a *plausible* justification regardless
of the real cause. That is fabricated memory.

**Consequence.** "Why did you buy ETH?" is answered from stored rows. If a field was not
stored, the answer says so.

**Status.** Active.

---

## D008 — `REAL_TRADING` raises rather than branches

**Decision.** `assert_paper_only()` raises `LiveTradingForbidden` when `REAL_TRADING=true`.

**Reason.** A conditional (`if settings.real_trading: ...`) means a `true` value silently
re-enables a path that does not exist, and a future contributor adds the path behind it. The
raise makes flipping the flag a loud, reviewable act.

**Consequence.** Intentional, not an oversight.

**Status.** Active.

---

## D009 — Laya runs via its own service, never LM Studio

**Decision.** Laya is a separate Python/PyTorch process on its own port (`laya-serve`, 8000).
It is not loaded as a GGUF model into LM Studio.

**Reason.** Verified from source: Laya is a ModernBERT encoder with a typed read-out head. It
is not a generative model and has no GGUF form.

**Consequence.** Two local processes. App uses port 8080 to avoid collision.

**Status.** Active.

---

## D010 — Run Laya on CPU by default

**Decision.** Laya runs on CPU; the GPU is reserved for the Main AI.

**Reason.** 8 GB VRAM is insufficient for both: Laya ~1.2 GB + an 8B Q4 model ~6.5 GB ≈
7.8 GB of 8 GB. Event volume does not justify GPU throughput for Laya.

**Consequence.** Benchmark actual latency in Phase 5 rather than assuming. If throughput is
insufficient, options are a smaller Main AI model or ONNX/TileLang acceleration — not
removing the CPU default blindly.

**Status.** Active. Benchmark pending (Phase 5).

---

## D011 — Treat order-flow features as hypotheses, not laws

**Decision.** No narrative claim ("price will hunt the liquidation cluster") enters the code
as a rule. Every feature must be measured as feature-X → outcome-Z before it gates a trade.

**Reason.** Untested market folklore is how systems acquire confident nonsense. Liquidation
"clusters" in particular are typically *modelled estimates* from OI, not observed levels.

**Consequence.** Features carry definitions (window, band, anchor, source) and are versioned.
Measurement tables live in [[25 - Research Log]].

**Status.** Active.

---

## D012 — Validation gates are not optimisation targets

**Decision.** Gates exist to reject nonsense, not to be optimised toward. Success requires
out-of-sample and walk-forward evidence, not passing a threshold on in-sample data.

**Reason.** Optimising to a gate guarantees a falsely good result.

**Consequence.** Variant counts are recorded; final holdout is touched once. Report negative
results with equal care.

**Status.** Active.

---

## D013 — The research loop proposes; humans apply

**Decision.** No self-modification of production code, strategy, or model. The loop writes
proposals to a research table only.

**Reason.** A model optimising against its own recent outcomes overfits, especially when most
individual losses are noise.

**Consequence.** Risk-rule changes are human-initiated even when the AI identifies pressure.

**Status.** Active.

---

## D014 — SQLite, single-user local

**Decision.** SQLite in WAL mode.

**Reason.** Single user on a desktop; zero administration; real relational integrity; single
file; standard SQL leaves a migration path.

**Consequence.** No multi-user scale. Acceptable and desirable — multi-user would add
attack surface for no benefit.

**Status.** Active.

---

## D015 — Store `answer_confidence`, never `confidence`, as a probability

**Decision.** Calibration and gating use Laya's `answer_confidence` only. The `confidence`
field (1 − normalised entropy) is never treated as a probability.

**Reason.** Verified from source: temperature scaling fits `answer_confidence`;
`confidence` is a concentration measure that shifts with option count and is not calibrated.

**Consequence.** Thresholds cannot be ported from other systems. See [[14 - Calibration]].

**Status.** Active.

---

## D016 — Laya context is the compact state, never the raw order book

**Decision.** Laya receives the state engine's derived summary. It never receives a raw L2
ladder, and it can never receive an image.

**Reason.** Verified: Laya is a text encoder with a 512-token budget on the English
checkpoint. A book ladder cannot fit, and `serialize_state` has no image or array path.

**Consequence.** The state engine must be terse and deterministic — which it should be
anyway.

**Status.** Active.

---

## D017 — Laya is pinned and re-verified before integration

**Decision.** Record the exact commit Laya integration is built against; re-verify the API
before Phase 5.

**Reason.** Laya is a 12-day-old project at time of writing with ~170 commits/week upstream.
Its API can change without notice.

**Consequence.** This vault documents Laya against commit `6d942c9` (v0.3.22). Upstream had
moved 174 commits ahead at the time of writing.

**Status.** Active. **Phase 5 must re-verify.**

---

## D018 — Jev is a reference, not a dependency

**Decision.** Laya is the decision engine. Jev-related architecture is referenced; no Jev
dependency is introduced.

**Reason.** The user selected Laya for local, free operation. Jev is a separate hosted
commercial product with its own API and cost.

**Consequence.** `DecisionEngine` remains an interface so a `JevDecisionEngine` is possible
later. The naming similarity between the two projects (both "System One", both "RLCD") is
noted for attribution clarity but is **not** a technical dependency.

**Status.** Active.

---

## Open decisions

| # | Question | Blocks |
|---|---|---|
| Q1 | Market data provider | Phase 3 |
| Q2 | Main LLM: local vs free remote | Phase 2 |
| Q3 | Spot vs perpetual as the primary instrument | Phase 3 |
| Q4 | Timeframe and symbol universe | Phase 3 |
| Q5 | Whether to fund historical order-book data | Phase 9 |

---

[[03 - System Architecture]] · [[27 - Known Limitations]] · [[30 - Coding Agent Instructions]] · [[25 - Research Log]]

## See also

- [[20 - Security]] — implementation of D001/D002
- [[09 - Risk Engine]] — implementation of D002/D003
- [[29 - Implementation Roadmap]] — where decisions become code