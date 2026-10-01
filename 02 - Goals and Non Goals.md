---
tags: [scope]
---

# 02 - Goals and Non Goals

Explicit scope boundaries. Anything outside these lists belongs in [[28 - Future Ideas]].

## Goals

### Conversation and reasoning
- Answer natural-language questions about markets, positions, and performance.
- Fetch current data through tools rather than recall.
- Explain past decisions from stored evidence, not reconstructed narrative.
- Refuse to agree with the user without independent verification.

### Market observation
- Price, candles, volume, order book, spread, depth, trades.
- Order-flow features: imbalance, volume delta, CVD, aggressor flow.
- Futures features where available: open interest, funding, liquidations.
- Every observation timestamped; staleness enforced.

### Structured decisions
- Laya provides fast, typed judgments on compact state.
- Main AI provides reasoning over Laya output and raw data.
- Both produce a structured trade proposal, not a string.

### Paper trading
- Simulated account, orders, fills, positions, fees, slippage.
- Realized and unrealized P&L, drawdown, exposure, equity.
- Complete, queryable history. Every trade explainable after the fact.

### Safety
- Deterministic risk engine between proposal and execution.
- Fail closed on every dependency.
- Kill switch no model can disable.
- Real-money execution absent from the codebase.

### Evidence
- Backtesting with explicit anti-leakage guarantees.
- Calibration measurement of Laya probabilities on project data.
- Train/validation/out-of-sample separation.
- No automatic promotion of a model or strategy to production.

### Transparency
- Dashboard showing what the AI did, tool by tool.
- Trade journal for every decision.
- Audit log for every consequential system event.

## Non-goals

| Not doing | Why |
|---|---|
| **Real-money trading** | Out of scope entirely. No code path exists. |
| **High-frequency / latency arbitrage** | Reference project `jev-trader` runs ~300ms per block on a specific L2. Wrong problem: this is research, not a market-making bot. |
| **Multi-user / auth / tenancy** | Single user on a desktop. Adding auth adds attack surface with no benefit. |
| **Cloud infrastructure** | Local-first. Internet APIs for market data only. |
| **Order-book replay from history** | Rarely available. Documented as a limitation rather than approximated. |
| **Automatic strategy promotion** | The research loop proposes; a human approves. |
| **Self-modifying production code** | The AI may propose changes; it never applies them. |
| **Multi-exchange support initially** | One provider behind an adapter. |
| **Profit optimisation before measurability** | Explicitly deprioritised. Correctness and reproducibility first. |

## Success criteria

The project succeeds when it is **measurable, testable and reproducible** — not when it
makes money. A system that honestly reports "this does not work" is a success. A system
that looks profitable because of a leak is a failure.

---

[[00 - Home]] · [[03 - System Architecture]] · [[28 - Future Ideas]]

## See also

- [[13 - Backtesting]] — integrity requirements
- [[26 - Decisions]] — why live trading is out of scope