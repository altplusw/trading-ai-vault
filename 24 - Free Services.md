---
tags: [services, cost]
---

# 24 - Free Services

## Purpose

Keep the project free-first. **Never hide recurring costs.** Every paid dependency is
optional, documented, and has a free alternative.

## Cost principles

1. Prefer local (Laya, SQLite) or keyless public APIs.
2. A paid tier is a **documented decision**, not an accident.
3. Any paid dependency must have a working free path.
4. No service is required for the core paper-trading loop.
5. Escalation budgets bound remote LLM spend ([[15 - Autonomous Trading]]).

## LLM providers

| Service | Free | Notes |
|---|---|---|
| **LM Studio (local)** | ✅ unlimited, free | Best default. No key, no cost, works offline. |
| DeepSeek | free tier | Generous; OpenAI-compatible. Good when local is too small. |
| Other OpenAI-compatible free tiers | varies | Adapter means swapping is config-only |
| Paid providers | n/a | Optional upgrade. Not required. |

Because `LLMProvider` is an interface ([[04 - Main AI]]), switching provider is a config
change, not a rewrite.

**Cost control:** the Main AI is the expensive component. That is the entire reason Laya
exists as a filter. Bound calls per symbol/hour, calls/day, and cost/day.

## Market data

| Service | Purpose | Free limit | Paid requirement | Free alternative |
|---|---|---|---|---|
| **Binance public REST/WS** | price, book, trades, funding, OI, liquidations | keyless, rate-limited | none | — |
| **CoinAPI** | market data | 10 req/s, 1000 req/day (Startup) | higher tiers for depth/history | Binance public |
| **CoinGlass** | OI, funding, liquidations | key required | **yes** | Binance funding/OI |
| **CoinMetrics** | institutional reference | community tier | higher tiers | Binance public |
| **Amberdata** | liquidation research | none | **yes** | skip the feature |
| **CoinGecko** | price/market cap | 10k/month | optional | Binance public |
| **ccxt** | multi-exchange library | n/a (library) | none | — |

⚠️ **Free-tier limits change.** Re-verify before relying on any row. Recorded here as of
2026-10-02.

**Probable paid needs**, flagged for the user's decision — not assumed:

| Need | Likely cost driver | Free workaround |
|---|---|---|
| Historical L2 order book | provider sells normalised snapshots | candle-based backtests, documented as such |
| Historical liquidations | paid aggregators | short window, or skip |
| Deep historical OI | paid tiers | recent window from Binance |

**Recommended: build the whole system on free/keyless Binance data first.** Add a paid
aggregator only if a specific, measured need appears.

## Software

| Component | Cost |
|---|---|
| Python, FastAPI, SQLite, pytest | free, open source |
| Laya | Apache-2.0 |
| `laya-serve` | free |
| Editor / tools | free |

## Hardware

No purchase required. The design targets the existing RTX 3060 Ti / i5-14400F / 32 GB.

**VRAM reality:** the Main AI and Laya together exceed 8 GB. Plan for **Laya on CPU**
([[05 - Laya]]). No GPU purchase needed.

## Recurring cost statement

**Current expected cost: $0.**

Everything required for paper trading, backtesting, the research loop, and Laya filtering
runs on free/local components. Costs appear only if the user chooses a paid LLM provider or
paid market data — both optional and both documented above.

## Testing

- No test may require a paid API or network access.
- The full test suite runs offline.
- A free-only configuration passes every safety gate.

---

[[06 - Market Data]] · [[04 - Main AI]] · [[22 - Deployment]] · [[23 - Configuration]]

## See also

- [[15 - Autonomous Trading]] — escalation budgets
- [[27 - Known Limitations]] — data gaps that may force a paid decision
- [[28 - Future Ideas]] — features that would add cost