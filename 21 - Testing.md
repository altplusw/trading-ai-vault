---
tags: [testing, quality]
---

# 21 - Testing

## Purpose

Demonstrate the system behaves as specified, **especially in failure**. A system that only
works when everything succeeds has not been tested.

## Principles

1. **Tests never touch the network, GPU, or model weights.** Live calls are not tests.
2. **Deterministic.** No wall-clock dependence, no unseeded randomness, no ordering assumptions.
3. **Test failure modes as first-class.** A failure test that passes because the code never
   reaches the failure branch is worthless.
4. **Never disable a test to make a build pass.** Fix the code or document why.
5. **Safety tests are non-negotiable.** They gate releases.

## Current state

`trading-ai/tests/` — **33 tests, passing**, no network, no GPU, no weights.

| File | Covers |
|---|---|
| `test_safety.py` | paper-only, `REAL_TRADING` raise, no exchange SDK, no credential refs, no order tools |
| `test_database.py` | schema, seeding, transactions, cascade, trace threading, persistence |
| `test_tools.py` | dispatch, unavailable shape, schema↔handler parity, autonomous flags |
| `test_laya.py` | fail-closed, protocol, answer validation, summarise |

## Required coverage by module

| Module | Must test |
|---|---|
| Market data parser | per-provider fixtures; malformed input; timestamp propagation |
| Timestamps | freshness detection; staleness cutoff; future-data rejection |
| State engine | determinism, purity, frozen output, truncation flags |
| Laya adapter | each question type; `score`→label conversion; unavailable; malformed |
| LLM adapter | each tool-call shape; unavailable; malformed args; round cap |
| Risk engine | every rule, pass **and** fail; **boundary values**; engine-exception ⇒ reject |
| Paper broker | fills, avg price, P&L, fees, slippage, limits, partials, cancellation |
| Orders / positions | invariants; cash never negative; exposure limits |
| P&L | hand-computed fixtures incl. fees |
| Backtest | look-ahead assertion; temporal splits; determinism |
| Calibration | Brier/ECE fixtures; baseline comparison; horizon required |
| Database | constraints; FK enforcement; transaction atomicity; idempotency |
| Autonomous mode | per-dependency failure halts loop; duplicate prevention; budgets |
| Kill switch | engaged ⇒ all rejects; persists restart; **not clearable by tool** |

## Failure scenarios — the required list

Each must have a test. These are the ones that matter most.

| Scenario | Expected |
|---|---|
| Laya unavailable | no autonomous trade |
| LLM unavailable | no autonomous trade |
| Market data unavailable | no trade; circuit opens |
| Database unavailable | loop halts |
| Stale price | rejected by risk (`stale_data:<field>`) |
| Invalid order | rejected at validation |
| Excessive size | rejected by risk |
| Daily loss limit hit | trading halted for day |
| Max drawdown hit | halted |
| Spread too wide | rejected |
| Missing data | rejected; `None` propagates |
| Duplicate order | idempotent no-op |
| Duplicate event | deduplicated |
| Kill switch engaged | everything rejected |
| Model supplies an extreme stop | risk engine overrides |

## Property-based tests

Randomised sequences should assert invariants, not specific outcomes:

- `cash >= 0` at all times.
- `exposure <= configured max`.
- No position without an opening fill.
- Sum of ledger deltas equals cash delta.
- Every position traces to complete trades.

Use Hypothesis for these where the dependency is acceptable.

## Look-ahead tests

The backtester's guarantee must itself be tested:

- A deliberately poisoned dataset (future values injected) must **raise**, not pass quietly.
- Train/validation timestamps strictly monotonic.
- Purge gap exists between splits.
- Intrabar ambiguity resolves to the stop.

## Safety gates (must pass before any release)

- `REAL_TRADING` raises.
- No exchange SDK importable.
- No credential-shaped source references.
- Kill switch engaged ⇒ every order type rejects, and no tool clears it.
- Broker rejects submission without a valid `risk_decision_id`.
- No order path exists that skips the risk engine.

## Commands

```bash
.\.venv\Scripts\python.exe -m pytest -q
```

CI runs the same command. No network access in CI.

## Coverage policy

No enforced percentage threshold. Target: every module has a test file; every rule has a
fail case; every external adapter has failure tests. **A coverage number does not demonstrate
correctness of trading logic** — the safety invariants above do.

---

[[20 - Security]] · [[13 - Backtesting]] · [[09 - Risk Engine]] · [[29 - Implementation Roadmap]]

## See also

- [[22 - Deployment]] — CI requirements
- [[24 - Free Services]] — why no test may depend on a paid API
- [[26 - Decisions]] — testing philosophy decisions