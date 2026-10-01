---
tags: [portfolio, positions]
---

# 11 - Portfolio and Positions

## Purpose

Present the account and its positions truthfully at all times, including mid-trade.

## Account state

Defined in [[10 - Paper Trading]]. Two rules:

1. **Equity is marked-to-market**, not cost-basis. `equity = available_cash + Σ |qty| × mark`.
2. **Marks come from the provider with a timestamp.** If no fresh mark exists, report
   equity as *stale* rather than as a clean number. A stale-but-confident equity is worse
   than an obviously-stale one.

## Positions

Per position: `symbol`, `side` (long-only v1), `quantity`, `avg_entry_price`, `mark_price`,
`unrealized_pnl`, `unrealized_pnl_pct`, `stop_price`, `target_price`, `opened_at`,
`strategy_id`, `source`, `decision_id`.

`stop_price` and `target_price` are displayed but **owned by the risk engine**, not the
strategy or the model. A model-suggested stop is advisory; the enforced stop is the risk
engine's.

## Derived metrics

| Metric | Definition | Trap to avoid |
|---|---|---|
| `unrealized_pnl` | (mark − entry) × qty | must use a fresh mark |
| `realized_pnl` | booked at close, net of fees | include fees or you overstate |
| `exposure` | Σ \|position value\| | **not** net — a hedged pair is still exposure |
| `concentration` | largest position ÷ equity | breaches a risk rule |
| `daily_pnl` | realised + unrealised today | UTC boundary must be explicit |
| `max_drawdown` | peak-to-trough on equity curve | needs a stored equity series |
| `win_rate` | wins ÷ closed trades | **meaningless without trade count** |
| `profit_factor` | gross win ÷ gross loss | undefined at zero loss |
| `expectancy` | mean P&L per trade | report alongside distribution |

## Honesty requirements

The spec is explicit: **never claim a profit the database does not confirm.**

- Report realised and unrealised **separately**. Conflating them is how systems
  manufacture the appearance of edge.
- Always show **trade count** next to any rate. "80% win rate" over 5 trades is noise.
- Show the **period** and the **account** the numbers describe.
- If marks are stale, label the whole panel stale. Do not quietly show last-known values
  as if current.
- Distinguish **paper** results from backtest results everywhere, without exception.

## Day boundary

`daily_pnl` resets at **00:00 UTC**. One definition, used by the risk engine, dashboard,
and reports. A local-time boundary would desynchronise the daily-loss limit from what the
user sees — a real source of confusion.

## Performance attribution

Group by: strategy version, symbol, model, Laya question set, source (manual/autonomous),
time of day. Report each group with sample size. This is how error analysis
([[16 - Self Improvement]]) finds what to fix.

## Equity curve

Store a point per decision and per fill, not only per day. Required for drawdown and for
any honest performance claim.

## Testing

- P&L arithmetic against hand-computed cases.
- Exposure is absolute, not net.
- Fees reflected in realised P&L.
- Stale marks ⇒ `stale: true`, never silent reuse.
- Win rate requires non-zero trade count; undefined states render honestly.
- Day boundary resets exactly at UTC midnight.
- Randomised sequences preserve invariants: cash ≥ 0, exposure ≤ limit, no phantom positions.

---

[[10 - Paper Trading]] · [[12 - Trade Journal]] · [[19 - UI Dashboard]] · [[13 - Backtesting]]

## See also

- [[09 - Risk Engine]] — how exposure and drawdown limits use these values
- [[17 - Database]] — where the equity series lives