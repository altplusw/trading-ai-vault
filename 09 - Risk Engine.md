---
tags: [risk, safety, core]
---

# 09 - Risk Engine

> **This is the load-bearing safety component.** If it can be bypassed, the entire safety
> model is theatre.

## Purpose

A **hard, deterministic boundary** between "a model proposed something" and "an order
exists." It approves or rejects. It never asks, never infers, never defers.

## Invariants

1. **No order reaches the broker without an approval carrying a `risk_decision_id`.**
2. **No model input reaches a limit.** Limits come only from configuration.
3. **Rejection is the default.** Approval requires passing every check.
4. **Fail closed.** Any internal error → reject.
5. **Every decision is explainable.** A rejection returns ordered reasons.
6. **No model can clear the kill switch.**
7. **Paper mode is the default** and real execution does not exist ([[20 - Security]]).

## Interface

```python
class RiskEngine(Protocol):
    def evaluate(self, proposal: TradeProposal, context: RiskContext) -> RiskDecision: ...
```

```python
@dataclass(frozen=True)
class RiskDecision:
    approved: bool
    risk_decision_id: str
    checks: tuple[RiskCheck, ...]      # ordered, every check present
    reasons: tuple[str, ...]           # populated when not approved
    sized_quantity: Decimal | None     # final size, after caps
    evaluated_at: float
```

The broker accepts **only** a proposal whose `RiskDecision.approved` is `True` and whose
`risk_decision_id` is present. That is enforced in the broker's signature, not by
convention.

## Check order

Every check runs; the decision is `not any(failed)`. Order is fixed so rejections are
deterministic and explainable:

1. **Mode** — is trading enabled? is it paper? is the kill switch engaged?
2. **Kill switch** — engaged ⇒ reject immediately, no further checks.
3. **Symbol** — on the whitelist?
4. **Data freshness** — every observation fresh enough? Freshness cutoff is per-field.
5. **Market sanity** — spread within limit; price positive; book coherent.
6. **Liquidity** — sufficient depth for intended size.
7. **Account state** — balance readable, equity computed, not inconsistent.
8. **Order size** — order ≤ max order % of equity.
9. **Position size** — resulting position ≤ max position % of equity.
10. **Concentration** — per-symbol and per-asset caps.
11. **Total exposure** — gross exposure ≤ max (leverage = 1 ⇒ ≤ 100%).
12. **Open positions** — count under limit.
13. **Daily loss** — daily P&L within limit ⇒ else halt for the day.
14. **Max drawdown** — within limit ⇒ else halt.
15. **Cooldown** — post-loss and post-abnormal-volatility cooldowns expired.
16. **Order frequency** — rate limit on new orders.
17. **Confidence** — if the proposal is model-derived, meets `min_confidence`. **Uses
    `answer_confidence` semantics from Laya, never `confidence`** ([[05 - Laya]]).
18. **Mode check** — autonomous vs manual rules (autonomous may be *more* restricted).

## Rule configuration

All from configuration, never from a model. Defaults in [[23 - Configuration]]:

| Rule | Default | Notes |
|---|---|---|
| `max_order_pct_equity` | 10% | |
| `max_position_pct_equity` | 20% | |
| `max_total_exposure_pct` | 100% | leverage 1 |
| `max_daily_loss_pct` | 5% | halts trading for the day |
| `max_drawdown_pct` | TBD | Phase 7 |
| `max_open_positions` | 10 | |
| `min_confidence` | 0.0 | **deliberately 0** — see note |
| `max_spread_bps` | TBD | venue-dependent, Phase 3 |
| `min_depth_multiple` | TBD | size must be ≪ visible depth |
| `stale_cutoff_seconds` | 30 | per-field |
| `loss_cooldown_seconds` | TBD | |
| `vol_cooldown_seconds` | TBD | |
| `max_orders_per_minute` | TBD | |
| `max_slippage_bps` | TBD | |
| `symbol_whitelist` | BTC/USDT, ETH/USDT, SOL/USDT | |
| `leverage` | 1 | disabled |
| `withdrawals` | forbidden | no code path |
| `autonomous_trading` | false | |

### Why `min_confidence` defaults to 0.0

Laya's shipped checkpoints are near chance on domain tasks ([[05 - Laya]]). Setting a
confidence floor against an uncalibrated model would reject everything or nothing
arbitrarily. Raise it **only after** measuring calibration on our own data
([[14 - Calibration]]). Until then, Laya is a filter, not a gate.

## Position sizing

Sizing is deterministic code. Nothing the model sends sets a size directly.

Order of operations:

1. Start from the strategy's proposed size (if any) or a configured default.
2. Cap by `max_order_pct_equity`.
3. Cap by `max_position_pct_equity` (accounting for the resulting position).
4. Reduce further if min-depth or slippage limits bind.
5. Reject if it rounds below a minimum viable size — never round *up*.

Volatility-adjusted sizing and capped-Kelly are research options ([[28 - Future Ideas]]). If
Kelly is ever used: verify calibration first, cap aggressively, use conservative inputs,
document the formula, and never let it exceed the hard caps above.

## Emergency stop

A distinct, higher-priority subsystem.

```python
class EmergencyStop:
    def engage(self, reason: str, flatten: bool = False) -> None
    def is_engaged(self) -> bool
    def requires_manual_clear(self) -> bool   # always True
```

On engage:

- Block **all** new orders immediately — checked at every gate.
- Cancel permitted open orders.
- Optionally flatten paper positions.
- Disable autonomous mode.
- Persist state (survives restart).
- Write an audit event with reason and timestamp.
- Clear requires **explicit human action** through the UI/API. No model path clears it.

Implementation must make it impossible to clear from the tool surface.

## Failure modes

| Failure | Behaviour |
|---|---|
| Risk engine raises | **Reject.** Never "assume allowed" |
| Account state unreadable | Reject |
| Data stale | Reject, reason `stale_data:<field>` |
| Provider circuit open | Reject |
| Kill switch engaged | Reject, reason `emergency_stop` |
| Proposal malformed | Reject at validation, before risk |

## Audit

Every evaluation persists: proposal id, `risk_decision_id`, each check with pass/fail and
observed vs limit values, verdict, reasons, resulting size, timestamp, config version.

A rejection must be reconstructible months later from the DB alone.

## Testing

- Each rule has a pass case and a fail case.
- **Boundary tests**: exactly at the limit passes, one unit over rejects.
- Kill switch: engaged ⇒ every order type rejects; cannot be cleared by any tool.
- Engine exception ⇒ reject.
- Rules come only from config — mutate config in a test, assert behaviour changes.
- **A model-supplied limit value is ignored.** Test: proposal contains `stop_loss` far away;
  engine overrides with its own configured rule.
- No order reaches broker without `risk_decision_id` (broker-level test).
- `REAL_TRADING=true` ⇒ system refuses to boot.

---

[[03 - System Architecture]] · [[10 - Paper Trading]] · [[20 - Security]] · [[21 - Testing]]

## See also

- [[18 - API and Tools]] — the Trade Proposal structure this engine receives
- [[23 - Configuration]] — every limit, with defaults
- [[26 - Decisions]] — why the model cannot override this