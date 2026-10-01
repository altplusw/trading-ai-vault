---
tags: [deployment, operations]
---

# 22 - Deployment

## Purpose

Run locally, reproducibly, on Windows, with clear startup order and honest failure.

## Target environment

- Windows 11
- Python 3.12 (managed by `uv`; Python ≥3.10 is the floor)
- RTX 3060 Ti 8 GB, i5-14400F, 32 GB RAM
- Runs entirely locally; internet used only for market data and (optionally) the LLM

## Components and their processes

| Component | Process | Port | Started by |
|---|---|---|---|
| Trading AI app | Python (uvicorn) | **8080** | user |
| Laya server | Python (`laya-serve`) | **8000** | user, separately |
| Space Bunny | LM Studio | 1234 | user, separately |
| Market data | outbound HTTPS | — | — |
| Database | SQLite file | — | app |

⚠️ **Port collision:** the app is on **8080** precisely because Laya claims **8000**. Do not
"simplify" this back to 8000.

Laya is **not** loaded into LM Studio. It is a PyTorch service, not a GGUF model.

## Setup

```bash
uv venv --python 3.12
uv pip install -e ".[dev]"
cp .env.example .env          # then edit
```

## Startup order

1. LM Studio — load a model, start the local server.
2. `laya-serve` — `LAYA_DEVICE=cpu LAYA_PRELOAD=1 laya-serve` (Phase 5+).
3. The app — `python -m uvicorn app.main:app --port 8080`.

The app **starts without 1 or 2** and reports them as offline. It degrades visibly rather
than failing to boot. Autonomous trading does not start until dependencies are healthy.

## Boot sequence (enforced in `app/main.py`)

1. `assert_paper_only()` — **refuse to boot** if `REAL_TRADING=true`.
2. `init_db()` — schema + seeded paper account.
3. Logging setup.
4. Yield (serve).

## Health

`GET /api/health` reports app mode, LLM reachability, Laya reachability, autonomous state.
The UI shows a dot per dependency. **Any dependency down ⇒ no autonomous trade.**

## Operational rules

- Bind `127.0.0.1`. Not a network service.
- Logs are JSON, one object per line, with `decision_id` for correlation.
- The SQLite file is the source of memory — back it up (`VACUUM INTO`).
- No cloud infrastructure. No containers required for local use.

## CI

- Runs `pytest -q` on the same command developers run.
- **No network, no GPU, no weights.**
- Safety gates are required checks; they cannot be skipped.
- Ruff/compile checks per `AGENTS.md` conventions.
- Python 3.10–3.13 matrix.

## Failure behaviour

| Failure | Behaviour |
|---|---|
| Config invalid | Refuse to boot, print the offending variable |
| DB unwritable | Refuse to boot |
| LLM down | App runs; chat reports it; no trading |
| Laya down | App runs; Laya tools report unavailable; no trading |
| Market data down | Circuit opens; trading halts; UI shows it |
| Disk full | Fail closed; refuse new orders |

## Not deployed

No cloud, no hosted DB, no container orchestration, no CI that touches real services. A
desktop research tool does not need them, and each would add failure surface.

---

[[23 - Configuration]] · [[19 - UI Dashboard]] · [[21 - Testing]] · [[05 - Laya]]

## See also

- [[24 - Free Services]] — external dependency costs
- [[20 - Security]] — network exposure rules
- `trading-ai/.env.example` — every variable