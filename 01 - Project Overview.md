---
tags: [overview]
---

# 01 - Project Overview

## Purpose

A local-first AI trading chatbot that operates a paper-trading account, explains its own
decisions from stored evidence, and never places a real order.

## What exists today

`trading-ai/` — a Phase 1 foundation. Honest inventory:

| Component | Status | Location |
|---|---|---|
| FastAPI app + lifespan boot | Implemented, tested | `app/main.py` |
| Configuration with safety invariants | Implemented, tested | `app/config.py` |
| Structured JSON logging | Implemented | `app/logging_setup.py` |
| SQLite schema (10 tables) | Implemented, tested | `database/schema.sql` |
| Repository / data access | Implemented, tested | `database/repo.py` |
| Main AI client (OpenAI-compatible) | Implemented | `llm/client.py` |
| Tool registry + 24 schemas | Implemented, tested | `llm/tools.py` |
| Laya HTTP client (fail-closed) | Implemented, tested | `laya/client.py` |
| Chat orchestration loop | Implemented | `app/chat.py` |
| Autonomous mode flag | Implemented, tested | `app/runtime.py` |
| Chat + portfolio UI | Implemented | `ui/index.html` |
| Test suite (33 tests) | Passing | `tests/` |

**Not implemented:** market data, paper broker, risk engine, backtester, calibration,
autonomous loop, kill switch, order-flow layer.

**By design not implemented:** live exchange execution. See [[20 - Security]].

## Verified working

- Server boots on `127.0.0.1:8080`, serves UI and JSON API.
- With the LLM down, chat returns a clear failure and records **zero** decisions.
- Autonomous mode toggles but cannot place anything.
- `REAL_TRADING=true` raises `LiveTradingForbidden` at startup.
- Tests assert no exchange SDK is importable and that source contains no
  credential-shaped references.

## Design intent

The implementation is a **skeleton in service of the documented architecture**, not a
product. Each phase in [[29 - Implementation Roadmap]] adds one component behind an
interface so that any component can later be replaced without rewriting the system.

## Relationship to the vault

This vault is the specification. Where code and documentation disagree, the documentation
is the specification and the code has a bug. Where the documentation is wrong or impossible,
update the documentation deliberately — record it in [[26 - Decisions]] — do not silently
diverge.

---

[[00 - Home]] · [[02 - Goals and Non Goals]] · [[03 - System Architecture]] · [[27 - Known Limitations]]

## See also

- [[29 - Implementation Roadmap]] — what to build next
- [[21 - Testing]] — how correctness is demonstrated