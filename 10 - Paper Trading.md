---
tags: [paper-trading, execution, core]
---

# 10 - Paper Trading

## Purpose

A deterministic **simulated exchange**. It is the only execution path in this project.

## Invariants

1. **No real-money execution exists.** No exchange SDK is installed or imported. A test
   asserts this ([[20 - Security]]).
2. **The broker accepts only risk-approved proposals.** Its entry point requires a
   `risk_decision_id`.
3. **All fills are simulated** from stored or supplied market data with modelled fees and
   slippage. There is no live fill source.
4. **Every state change is journalled** and reconstructable from the DB.

## Interface

```python
class Broker(Protocol):
    def submit(self, proposal: TradeProposal, approval: RiskDecision) -> Order: ...
    def cancel(self, order_id: str) -> Order: ...
    def positions(self) -> list[Position]: ...
    def account(self) -> AccountState: ...
```

## Account model

Spot-style, USDT-quoted, leverage 1. Fields tracked:

| Field | Definition |
|---|---|
| `cash` | settled quote balance |
| `reserved` | held for resting limit orders |
| `available_cash` | `cash − reserved` |
| `equity` | `available_cash + Σ position market value` |
| `unrealized_pnl` | marked against current price |
| `realized_pnl` | booked on close |
| `fees_paid` | cumulative |
| `exposure` | Σ abs(position value) |
| `daily_pnl` | realised + unrealised, reset on UTC day boundary |
| `open_positions`, `open_orders` | counts |

All `Decimal`, never `float`. Rounding is explicit and centralised.

## Order lifecycle

```
PENDING ──fill──► PARTIAL ──fill──► FILLED
   │                  │
   └──cancel──► CANCELLED        (any time before FILLED)
   
   └──risk/validation failure──► REJECTED
```

Order fields (per spec): `id`, timestamps, `symbol`, `side`, `order_type`, requested
quantity/price, executed quantity/price, `fee`, `slippage`, `status`, `source`,
`strategy_id`, `reason`, `model_decision_id`, `laya_decision_id`, `risk_decision_id`.

## Fill simulation

Deterministic given inputs. Document every assumption.

**Market orders** — fill at mid plus configured slippage, plus fee.

**Limit orders** — rest. Fill when a supplied price crosses the limit:

- Buy limit fills when `price ≤ limit`.
- Sell limit fills when `price ≥ limit`.
- Resting orders reserve cash/size at placement.

**Slippage model** — configurable `slippage_bps`, applied against the trade direction.
Depth-aware slippage is a research refinement; if modelled, document the depth function.

**Fees** — `maker_fee_bps` for resting, `taker_fee_bps` for immediate. Charged on notional
at fill, in quote currency.

**Partial fills** — supported by the model. In live-mode simulation, a limit order crossing
by a small amount fills partially; the remainder rests. Document the exact rule.

**No intrabar sequence knowledge.** If a candle's high *and* low cross a stop and a target,
the **order of events is unknowable** from OHLC. Policy: assume the **worse** outcome
(stop first) unless intrabar data is available. This is a conservative bias that must be
documented in every backtest report.

## Positions

- One net position per symbol, spot-style: no shorting in v1.
- Average entry price updated with each fill (weighted).
- Long only until shorting is explicitly designed and tested.
- Realised P&L booked to cash and to `positions.realized_pnl` on reduce/close.
- Fees deducted at every fill.

## Order sources and why it matters

`USER` · `AUTONOMOUS_AI` · `BACKTEST` · `MANUAL`.

Recorded on every order so performance can be attributed. Autonomous results are never
silently mixed with manual ones.

## Isolation from live trading

- The `Broker` interface has one implementation: `PaperBroker`.
- No module imports an exchange SDK. Asserted by test.
- `REAL_TRADING` cannot be enabled ([[20 - Security]]).
- If live trading is ever added, it is a **new broker implementation** behind the same
  interface, in a separate package, gated by explicit human authorization
  ([[29 - Implementation Roadmap]] Phase 14). It is not a flag flip.

## Determinism

Same inputs ⇒ same outputs. No wall-clock dependence in fill logic (time comes from the
data), no unseeded randomness, no floating-point accumulation.

## Testing

- Buy/sell fills, average entry price, realised P&L on partial and full closes.
- Fees at maker and taker rates; fee deducted from cash correctly.
- Slippage applied in the correct direction.
- Limit order rests, reserves, fills on cross, cancels cleanly.
- Cash never negative; insufficient balance ⇒ reject.
- Position/exposure/equity invariants after randomised sequences (property-based).
- Broker rejects any submission lacking a valid `risk_decision_id`.
- Every fill writes ledger + journal + audit rows.
- Determinism: identical input sequence ⇒ identical final state.

---

[[09 - Risk Engine]] · [[11 - Portfolio and Positions]] · [[12 - Trade Journal]] · [[13 - Backtesting]]

## See also

- [[17 - Database]] — orders, fills, positions, ledger tables
- [[14 - Calibration]] — measuring outcomes against Laya probabilities
- [[20 - Security]] — why live execution is absent