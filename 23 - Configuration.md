---
tags: [configuration]
---

# 23 - Configuration

## Purpose

All behaviour is configuration, not code. In particular: **risk limits come only from
configuration and never from a model** ([[09 - Risk Engine]]).

## Principles

- Environment variables, loaded from `.env`.
- `.env.example` is committed. **`.env` is never committed.**
- No secret ever has a default. A missing required secret ⇒ fail closed at startup.
- Config is **immutable for the process lifetime** and versioned; a change is an audited
  event. Rules cannot change mid-session silently.
- Every limit has a documented default and a rationale.

## Variables

Authoritative source: `trading-ai/.env.example`.

### Main AI

| Variable | Default | Notes |
|---|---|---|
| `LLM_PROVIDER` | `openai_compatible` | provider adapter |
| `LLM_BASE_URL` | `http://localhost:1234/v1` | any OpenAI-compatible server |
| `LLM_API_KEY` | `lm-studio` | often a placeholder locally |
| `LLM_MODEL` | *(blank)* | blank ⇒ auto-discover |
| `LLM_TEMPERATURE` | `0.4` | |
| `LLM_MAX_TOKENS` | `1200` | |
| `LLM_TIMEOUT_SECONDS` | `120` | |

**`LLM_MODEL` is never hard-coded.** The whole point is swappability ([[04 - Main AI]]).

### Laya

| Variable | Default | Notes |
|---|---|---|
| `LAYA_URL` | `http://127.0.0.1:8000` | its own process |
| `LAYA_TIMEOUT_SECONDS` | `15` | |
| `LAYA_MIN_CONFIDENCE` | `0.0` | **deliberately 0** until calibrated |

### Market data

| Variable | Default | Notes |
|---|---|---|
| `MARKET_DATA_PROVIDER` | `none` | no provider chosen yet |
| `DEFAULT_SYMBOL` | `BTC/USDT` | |
| `MARKET_DATA_MAX_STALENESS_SECONDS` | `30` | fail closed above this |

### Paper trading

| Variable | Default | Notes |
|---|---|---|
| `PAPER_STARTING_BALANCE` | `10000` | |
| `PAPER_QUOTE_CURRENCY` | `USDT` | |
| `PAPER_TAKER_FEE_BPS` | `10` | 0.10% |
| `PAPER_MAKER_FEE_BPS` | `5` | 0.05% |
| `PAPER_SLIPPAGE_BPS` | `5` | 0.05% |

Fees and slippage are **assumptions**, not measured values. Every result must state them
([[10 - Paper Trading]]).

### Safety

| Variable | Default | Notes |
|---|---|---|
| `REAL_TRADING` | `false` | **raises at startup if true** |
| `AUTONOMOUS_TRADING` | `false` | |

### Risk limits

| Variable | Default | Notes |
|---|---|---|
| `RISK_MAX_POSITION_PCT` | `20` | % equity |
| `RISK_MAX_ORDER_PCT` | `10` | % equity |
| `RISK_MAX_TOTAL_EXPOSURE_PCT` | `100` | leverage 1 |
| `RISK_MAX_DAILY_LOSS_PCT` | `5` | halts for the day |
| `RISK_MAX_OPEN_POSITIONS` | `10` | |
| `RISK_MIN_CONFIDENCE` | `0.0` | **0 until calibrated** |
| `RISK_SYMBOL_WHITELIST` | `BTC/USDT,ETH/USDT,SOL/USDT` | |

To be added: `RISK_MAX_DRAWDOWN_PCT`, `RISK_MAX_SPREAD_BPS`, `RISK_MIN_DEPTH_MULTIPLE`,
`RISK_LOSS_COOLDOWN_SECONDS`, `RISK_VOL_COOLDOWN_SECONDS`, `RISK_MAX_ORDERS_PER_MINUTE`,
`RISK_MAX_SLIPPAGE_BPS`.

### Server

| Variable | Default | Notes |
|---|---|---|
| `SERVER_HOST` | `127.0.0.1` | |
| `SERVER_PORT` | `8080` | **not 8000** — Laya's port |
| `LOG_LEVEL` | `INFO` | |

## Why several defaults are deliberately permissive

`LAYA_MIN_CONFIDENCE = 0.0` and `RISK_MIN_CONFIDENCE = 0.0` are **intentional**, not
oversights.

Laya's shipped checkpoints are near chance on domain tasks and fail confidently
([[05 - Laya]]). A confidence floor against an uncalibrated model would reject
arbitrarily — it would look like a safeguard while actually providing none.

Set them only after [[14 - Calibration]] shows real skill, and document the value and the
evidence behind it in [[25 - Research Log]]. A permissive default that is *documented as
permissive* is honest; a permissive default that pretends to be protective is not.

## Validation

At startup:

- Parse and type-check every variable.
- Reject unknown values (e.g. a provider name that does not exist).
- Reject out-of-range numbers with a message naming the variable.
- Require secrets to be present when the selected provider needs them.
- Refuse to boot on `REAL_TRADING=true`.
- Log the **effective** configuration (redacted) for reproducibility.

## Config versioning

Every decision records the config version in force. Without this, a later results review
cannot tell which rules produced them.

## Testing

- Each variable's default is asserted.
- Out-of-range values fail with a message naming the variable.
- Unknown provider is rejected at startup.
- Changing a risk limit changes behaviour (proving limits come from config, not code).
- `REAL_TRADING=true` refuses to boot.
- Secrets absent ⇒ fail closed.

---

[[20 - Security]] · [[09 - Risk Engine]] · [[24 - Free Services]] · [[22 - Deployment]]

## See also

- `trading-ai/app/config.py` — the implemented loader and `assert_paper_only`
- `trading-ai/.env.example` — authoritative list
- [[26 - Decisions]] — the permissive-confidence decision