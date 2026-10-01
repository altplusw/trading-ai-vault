---
tags: [market-data, core, open-question]
---

# 06 - Market Data

> ⚠️ **Open question — read first.** "OpenMarket" **could not be verified** to exist as a
> public crypto market-data API. Searches returned CoinAPI, Amberdata, CoinMetrics,
> CoinGlass, CoinGecko, CoinMarketCap, ccxt, Massive, Tiingo — but no OpenMarket.
> **No OpenMarket endpoints are documented here, because none were verified.**
> See [[27 - Known Limitations]].

## Purpose

Provide raw, timestamped market observations to the Market State Engine. This layer fetches
and normalises; it **does not** compute indicators or strategy features.

## Provider interface

```python
class MarketDataProvider(Protocol):
    def get_ticker(self, symbol: str) -> Ticker: ...
    def get_candles(self, symbol: str, timeframe: Timeframe, limit: int) -> list[Candle]: ...
    def get_order_book(self, symbol: str, depth: int) -> OrderBook: ...
    def get_recent_trades(self, symbol: str, limit: int) -> list[Trade]: ...
    def get_funding_rate(self, symbol: str) -> FundingRate | None: ...
    def get_open_interest(self, symbol: str) -> OpenInterest | None: ...
    def get_liquidations(self, symbol: str, window: timedelta) -> list[Liquidation] | None: ...
    def get_symbols(self) -> list[Symbol]: ...
    def get_market_status(self) -> MarketStatus: ...
    def health(self) -> ProviderHealth: ...
```

Optional methods return `None` when the venue does not support them. **Unsupported is not
the same as zero.** A provider must never fabricate a value for a capability it lacks —
see [[07 - Order Flow]].

## Provider selection criteria

Scored before choosing:

1. **Cost** — free tier must cover our cadence. Free-first ([[24 - Free Services]]).
2. **Historical depth** — backtesting needs history ([[13 - Backtesting]]).
3. **Historical order book / trade-level** — needed for order-flow backtests. Usually paid.
4. **Perp data** — funding, OI, liquidations require a derivatives venue.
5. **Rate limits** — must sustain the poll cadence.
6. **Auth** — keyless preferred for a local single-user tool.
7. **Reliability** — uptime, and whether it can fail closed.

## Verified candidate providers

Researched 2026-10-02. **Free-tier details change — re-verify before committing.**

| Provider | Free | Order book | OI / funding | Liquidations | Historical order book | Notes |
|---|---|---|---|---|---|---|
| **Binance public REST/WS** | Yes, keyless | ✅ L2 depth | ✅ | ✅ streams | ⚠️ limited | Best free real-time crypto. Strong default candidate. |
| **CoinAPI** | 10 req/s, 1000/day (Startup) | ✅ | via reference rates | ⚠️ | ⚠️ normalized L2 snapshots, downsampled — explicitly **not** bit-identical to live | Key required |
| **CoinGlass** | ❌ key required, paid | ⚠️ | ✅ OI + funding | ✅ | ✅ | Aggregator; strong for OI/funding/liquidation |
| **CoinMetrics** | community tier | ⚠️ | ✅ OI streams | ⚠️ | ✅ | `CM_API_KEY`; good institutional reference |
| **Amberdata** | ❌ paid | ✅ | ✅ | ✅ | ✅ | Purpose-built for liquidation research |
| **ccxt** | library | ✅ many venues | venue-dependent | venue-dependent | venue-dependent | **Unified adapter library** — best insurance against provider lock-in |

### Recommendation

**Start with Binance public endpoints via a thin adapter** (free, keyless, best real-time
coverage). Keep `ccxt` as a fallback for portability. Use CoinGlass/CoinAPI later if
historical OI/liquidation coverage proves necessary and a budget is approved.

If data must be swappable later, the interface above is the seam. Do not adopt a provider
before writing the adapter.

## Symbol convention

Internal canonical form: `BASE/QUOTE`, e.g. `BTC/USDT`. Providers use their own formats
(`BTCUSDT`, `BTC-USDT`, `XBTUSD`). **Mapping lives in the adapter.** The rest of the
application never sees a vendor symbol format.

Perp contracts are distinct instruments. If OI/funding is used, decide explicitly whether
a symbol means spot or a specific perpetual (e.g. `BTC/USDT` vs `BTC/USDT:USDT-PERP`) and
document it. Mixing spot and perp data silently is a correctness bug.

## Timestamps — non-negotiable

Every observation carries a timestamp and a **source**. Rules:

- Provider timestamp preferred. If absent, record `source_ts = None` and treat as **stale**
  until verified — never substitute local receive time as if it were market time.
- Local `received_at` recorded separately.
- All comparisons and staleness checks use `source_ts`.
- Stored snapshots are immutable.

Full rules and the look-ahead guarantees they support: [[13 - Backtesting]] §Integrity,
[[21 - Testing]].

## Timeframes

Define explicitly; do not inherit provider naming.

| TF | Duration |
|---|---|
| `1m` `5m` `15m` `1h` `4h` `1d` | minutes / hours / days |

Candles are **closed candles only** for signal generation. An in-progress candle is not a
fact; using it in a backtest is look-ahead. The state engine must be able to request
closed-candle-only data.

## Rate limiting and politeness

- Per-provider token bucket / min-interval.
- Respect `Retry-After` and provider backoff headers.
- Circuit-breaker: N consecutive failures → provider marked unhealthy → **no trade**
  (fail closed).
- Log every request: provider, endpoint, symbol, latency, outcome, rate-limit state.
- Cache historical candles locally; never re-fetch history already stored.

## Failure cases

| Case | Behaviour |
|---|---|
| Provider unreachable | Circuit opens; system halts trading; UI shows it |
| Rate limited | Back off, honour `Retry-After`; halt if unrecoverable |
| Stale data | Rejected by risk engine ([[09 - Risk Engine]]) |
| Unsupported feature | Return `None`; feature disabled; never substituted |
| Malformed response | Log, treat as failure, never partially parse |
| Symbol not listed | Explicit error, not an empty result |

## Testing

- Parser tests per provider against recorded fixtures (never live calls in tests).
- Timestamp propagation and staleness detection.
- Adapter normalises vendor symbols and shapes.
- Unsupported-feature path returns `None`.
- Malformed-response path raises rather than half-parsing.

---

[[03 - System Architecture]] · [[07 - Order Flow]] · [[13 - Backtesting]] · [[27 - Known Limitations]]

## See also

- [[24 - Free Services]] — cost comparison
- [[29 - Implementation Roadmap]] — Phase 3
- [[21 - Testing]] — parser and timestamp tests