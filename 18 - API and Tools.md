---
tags: [api, tools, mcp]
---

# 18 - API and Tools

## Purpose

Expose the system to the Main AI and the user through **explicit, validated, typed tools**,
so the model retrieves facts instead of recalling them.

## Two interfaces

| Interface | Audience | Transport |
|---|---|---|
| **Tool layer** | Main AI (model-invoked) | in-process function call |
| **HTTP API** | UI, scripts, external clients | REST/JSON over localhost |

Both call the **same underlying services**. No logic lives in a route handler that is not
also reachable from a tool — otherwise the model and the user see different behaviour.

## Tool contract

Every tool:

1. Takes validated, typed arguments. **Never trust model input.**
2. Returns JSON-serialisable data.
3. Returns **errors as data**, not exceptions:
   `{"ok": false, "error": "...", "detail": "...", "guidance": "..."}`
4. Never executes an order without a risk decision id.
5. Is side-effect-free unless explicitly documented as mutating.
6. Records an audit event if it changes anything.

`guidance` tells the model how to speak about the failure — specifically, **not** to guess.

## Tool catalogue

Implemented (`trading-ai/llm/tools.py`) vs planned:

| Tool | Status | Notes |
|---|---|---|
| `get_account` | ✅ | paper balance, mode |
| `get_positions` | ✅ | |
| `get_open_orders` | ✅ | |
| `get_trade_history` | ✅ | |
| `get_trade_journal` | ✅ | supports `decision_id` |
| `get_decision_history` | ✅ | |
| `create_trade_plan` | ✅ | records intent, **executes nothing** |
| `get_autonomous_status` | ✅ | |
| `start_autonomous_mode` | ✅ | flag only (Phase 11 adds the loop) |
| `stop_autonomous_mode` | ✅ | |
| `check_system_health` | ✅ | |
| `get_market_price` | ⬜ Phase 3 | |
| `get_candles` | ⬜ Phase 3 | |
| `get_order_book` | ⬜ Phase 3 | |
| `get_market_stats` | ⬜ Phase 3 | |
| `get_order_flow` | ⬜ Phase 4 | |
| `get_liquidation_data` | ⬜ Phase 4 | optional/venue-dependent |
| `get_open_interest` | ⬜ Phase 4 | optional/venue-dependent |
| `get_funding` | ⬜ Phase 4 | optional/venue-dependent |
| `ask_laya` | ⬜ Phase 5 | |
| `validate_trade_plan` | ⬜ Phase 7 | |
| `paper_buy` / `paper_sell` | ⬜ Phase 6 | **behind risk engine** |
| `cancel_order` / `close_position` | ⬜ Phase 6 | |
| `get_pnl` / `get_performance` | ⬜ Phase 6/7 | |
| `run_backtest` | ⬜ Phase 9 | |
| `pause_autonomous_mode` / `resume_autonomous_mode` | ⬜ Phase 11 | |
| `emergency_stop` | ⬜ Phase 7 | **human-only — see below** |

### `emergency_stop` is special

Engaging the kill switch is **not a model tool**, or if exposed, it is **engage-only** and
can never be cleared through the tool surface. Clearing requires the UI. This prevents a
model from toggling the safety boundary off ([[09 - Risk Engine]]).

Currently unimplemented tools are declared in the schema but return
`{"ok": false, "error": "unavailable", "implemented_in": "Phase N"}` — so the model knows
the intended surface and reports the gap honestly instead of inventing data. **A test
asserts every declared schema has a handler** (a missing handler is a defect).

## Trade proposal

The structured object the risk engine receives.

| Field | Notes |
|---|---|
| `symbol`, `direction` | |
| `entry_type`, `entry_price` | |
| `stop_loss`, `take_profit` | **advisory**; the risk engine may override |
| `position_size`, `risk_amount` | sizing re-derived by risk engine, never trusted |
| `timeframe`, `thesis`, `invalidation` | |
| `market_state` | the exact `FeatureSet` |
| `laya_decisions` | raw output |
| `confidence` | `answer_confidence` semantics |
| `expected_conditions` | |
| `strategy_id` + version, `model_id`, `laya_version` | |
| `timestamp`, `decision_id` | |

⚠️ **The AI cannot submit arbitrary order parameters.** Size, stop, and limits are
re-derived deterministically by the risk engine. Model values are inputs to be checked, not
values to be obeyed.

## HTTP API

Implemented (Phase 1):

| Route | Purpose |
|---|---|
| `GET /` | chat + portfolio UI |
| `GET /api/health` | app, LLM, Laya, mode |
| `POST /api/chat` | one conversational turn |
| `GET /api/conversations` | list |
| `GET /api/conversations/{id}/messages` | history |
| `GET /api/portfolio` | account, positions, orders |
| `GET /api/decisions` | decision history |
| `GET /api/journal` | journal |
| `GET /api/autonomous` | mode status |
| `POST /api/autonomous/{start\|stop}` | mode control |

Planned: `/api/risk/rules`, `/api/orders`, `/api/positions`, `/api/backtests`,
`/api/calibration`, `/api/research`, `/api/kill-switch`.

## Safety properties of the HTTP layer

- Binds `127.0.0.1` by default. **Not exposed to a network.**
- If an API key is configured, bearer auth with **constant-time** comparison on bytes.
- Body size, question count, option count, and concurrency limits enforced **before**
  parsing completes where possible.
- `500` responses never leak tracebacks, paths, or internals to the client; details go to
  the log.
- The kill switch route requires a human session and cannot be invoked by a model.

## Audit

Every tool call is logged with name, arguments (redacted), result status, and duration.
Mutating tools additionally write audit events ([[20 - Security]]).

## Testing

- Every tool rejects malformed arguments.
- Unavailable tools return the documented error shape, never partial data.
- Mutating tools write audit events.
- `paper_buy`/`paper_sell` unavailable until Phase 6, and even then require a risk decision.
- HTTP auth: missing/invalid key ⇒ 401; comparison is constant-time.
- Model cannot clear the kill switch via any route.
- Tool results are JSON-serialisable (catches numpy/Decimal leakage).

---

[[04 - Main AI]] · [[09 - Risk Engine]] · [[19 - UI Dashboard]] · [[20 - Security]]

## See also

- `trading-ai/llm/tools.py` — implemented registry
- [[03 - System Architecture]] — where tools sit in the trust chain
- [[21 - Testing]] — tool-layer test requirements