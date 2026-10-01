---
tags: [home, index]
---

# 00 - Home

## Project purpose

A local-first AI trading chatbot that operates a **paper-trading** account. You talk to it;
it looks at the market, reasons, proposes trades, and places simulated orders under
deterministic risk control. It remembers every decision and can explain why it acted.

## Current status

| | |
|---|---|
| **Current phase** | **Phase 0 — Documentation** |
| **Next phase** | Architecture review (Gate 2) |
| **Implementation exists** | Phase 1 skeleton only (`trading-ai/`) — see [[01 - Project Overview]] |
| **Paper trading** | Not implemented |
| **Live trading** | **Does not exist and is not planned** |

A working Phase 1 foundation exists: FastAPI app, SQLite schema, Space Bunny
(LLM) connection, 24-tool registry, chat UI, 33 passing tests. It **cannot place a trade** —
the execution tools are schema-only stubs by design.

## Visual maps

- **[Trading AI - System Map.canvas](Trading%20AI%20-%20System%20Map.canvas)** \u2014 the architecture as clickable cards
- **[Trading AI - Roadmap Map.canvas](Trading%20AI%20-%20Roadmap%20Map.canvas)** \u2014 phases, gates, and blockers

## Architecture

```
        USER
          │
    CHAT UI / DASHBOARD
          │
    MAIN TRADING AI  ← conversational, reasons, proposes
          │
     TOOL LAYER
   ↙      ↓       ↘
MARKET   LAYA   PORTFOLIO
DATA   (local)    TOOLS
   ↘      ↓       ↙
   TRADE PROPOSAL  (structured, typed)
          │
   DETERMINISTIC RISK ENGINE   ← hard boundary, no model input
          │
     PAPER BROKER              ← simulated execution
          │
       DATABASE                ← memory
          │
   ┌──────┴──────┐
   ▼             ▼
TRADE      BACKTEST /
JOURNAL    RESEARCH LOOP
```

Full detail: [[03 - System Architecture]]

## Core components

| Component | Role | Deterministic? |
|---|---|---|
| **Main Trading AI** | Conversation, reasoning, tool selection, trade proposals | No — model output |
| **Laya** | Fast local structured judgments, filtering | No — model output |
| **Market State Engine** | Raw data → compact immutable snapshot | **Yes** |
| **Risk Engine** | Decides what is allowed | **Yes** |
| **Paper Broker** | Simulated fills, positions, P&L | **Yes** |
| **Database** | Memory, audit, training data | **Yes** |
| **Backtester** | Evidence | **Yes** |

## The one principle

> **MODELS ADVISE. CODE DECIDES.**

A model may propose an action. Only deterministic code may validate and execute it.
No model can widen a risk limit, bypass a check, or disable the kill switch.
See [[09 - Risk Engine]] and [[26 - Decisions]].

## Safety model

- `REAL_TRADING=false` by default and **cannot be enabled** — startup raises rather than branches.
- Every trade proposal passes the risk engine. Rejections are deterministic and explainable.
- **Fail closed**: LLM down, market data stale, Laya down, database down → **no trade**.
- No exchange SDK is installed or imported. No withdrawal-capable credential is ever read.
- Emergency stop is not controllable by any model.
- Model output is never treated as a fact; the chatbot is instructed to say when it could not
  get data rather than guessing.

Detail: [[20 - Security]] · [[09 - Risk Engine]]

## Roadmap

[[29 - Implementation Roadmap]] — 15 phases, 10 approval gates. Live trading is Phase 14
and requires explicit human authorization. It is not on the critical path.

## Important decisions

Summarised in [[26 - Decisions]]. The load-bearing ones:

1. **Paper only.** No real-money path in this codebase.
2. **Models advise, code decides.** Deterministic risk gate between proposal and execution.
3. **Every component replaceable.** LLM provider, decision engine, and market data provider
   all sit behind interfaces.
4. **Laya is a cost filter first, a signal second.** Its shipped weights have never seen
   market data — see [[05 - Laya]] and [[27 - Known Limitations]].
5. **Fail closed everywhere.**
6. **Evidence before deployment.** Backtest → out-of-sample → paper → *then* consider anything else.

## Known limitations

Read [[27 - Known Limitations]] before trusting anything. The headline ones:

- **Laya's shipped checkpoints are near chance on domain tasks** (0.362 vs a 0.461 majority
  baseline on their own benchmark). Do not treat its output as a trading signal until fine-tuned
  on labelled data.
- **No market data provider is chosen or verified yet.** "OpenMarket" could not be confirmed
  to exist as a public API — see [[06 - Market Data]].
- **Laya is a text encoder.** It cannot read charts or images.
- No backtest, no calibration measurement, no autonomous loop, no live trading.

---

[[01 - Project Overview]] · [[03 - System Architecture]] · [[26 - Decisions]] · [[29 - Implementation Roadmap]]