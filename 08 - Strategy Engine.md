---
tags: [strategy, research]
---

# 08 - Strategy Engine

## Purpose

Define, version, and evaluate strategies as **documented, testable rules**. The LLM may
propose strategies; only the backtester decides whether one performs acceptably.

## Strategy definition

Every strategy is a versioned record. A strategy without complete metadata is not
registered.

| Field | Purpose |
|---|---|
| `strategy_id` | stable identifier |
| `version` | immutable; changes create a new version |
| `name`, `purpose` | what and why |
| `market`, `timeframe`, `symbols` | scope |
| `entry_condition` | deterministic predicate over `FeatureSet` |
| `exit_condition` | deterministic predicate |
| `stop_condition`, `take_profit_condition` | risk exits |
| `position_sizing_rule` | how size is derived (risk is applied separately) |
| `invalidation_condition` | what would prove the thesis wrong |
| `filters` | regime/microstructure gates |
| `required_data` | features it needs; missing ⇒ strategy disabled |
| `laya_questions` | which Laya questions it consults |
| `risk_requirements` | additional constraints on top of global rules |
| `status` | `draft` → `backtested` → `paper` → `retired` |

**A strategy may only reach `paper` after passing the validation gates**
([[13 - Backtesting]] §Gates). Status transitions are explicit and audited.

## Determinism requirement

Entry, exit, sizing and filter logic are **code**, not model output, wherever practical.
A strategy whose entry is "the LLM said so" is not a strategy — it is an experiment.
Record both:

- `entry_logic`: deterministic (required for backtest comparability)
- `laya_context`: model judgments recorded alongside, used for analysis and gating only
  once calibrated

This separation lets a backtest reproduce exactly, and lets us measure what Laya
contributed versus what the rules contributed.

## Strategy is not an edge

Registering a strategy asserts nothing about profitability. `status=draft` means exactly
"untested." Never label a strategy "profitable" — report measured metrics with their sample
sizes and confidence intervals, and state the validation set ([[13 - Backtesting]]).

## Registry

```python
class StrategyRegistry(Protocol):
    def get(self, strategy_id: str, version: int) -> StrategyDef: ...
    def register(self, draft: StrategyDef) -> StrategyDef: ...
    def list_by_status(self, status: StrategyStatus) -> list[StrategyDef]: ...
```

Immutable versions. A running strategy pins its version; editing means registering a new
version so historical results remain attributable.

## Interaction with Laya

- Laya answers a **fixed battery** defined by the strategy, not ad-hoc questions invented
  per trade. Ad-hoc questions cannot be calibrated or compared.
- Adding a question to the battery is a **strategy change** → new version → revalidate.
- A strategy may depend on Laya for *filtering* (skip a trade) even when uncalibrated.
  It may depend on Laya for *direction* only after calibration proves it better than the
  rule it would replace. See [[14 - Calibration]].

## First strategy

Deliberately simple and fully deterministic, so the harness itself is validated before
any model logic is added:

- **S0 — Baseline**: enter on a simple documented trigger, fixed risk sizing, symmetric stop.
- **S1 — Filtered baseline**: S0 + Laya regime filter (skip trades in unfavoured regime).
- **S2 — Flow-augmented**: S1 + a small set of order-flow features from [[07 - Order Flow]].

Comparing S0 → S1 → S2 out-of-sample is how we learn whether Laya and order flow add
anything. If they don't, that is a valid and useful result ([[25 - Research Log]]).

## Testing

- Strategy definitions validate (all fields present, entry/exit well-formed).
- Version immutability: editing raises.
- Status transitions require evidence of the prior gate.
- Entry/exit predicates are pure functions of a `FeatureSet`.
- Determinism: same input ⇒ same decision, always.
- `required_data` missing ⇒ strategy reports disabled rather than guessing.

---

[[07 - Order Flow]] · [[13 - Backtesting]] · [[14 - Calibration]] · [[16 - Self Improvement]]

## See also

- [[17 - Database]] — strategy/version tables
- [[26 - Decisions]] — why the LLM may propose but not apply
- [[25 - Research Log]] — measured results per strategy version