---
tags: [future, ideas]
---

# 28 - Future Ideas

## Purpose

A holding area so scope creep becomes a **recorded idea** rather than silent expansion.
Nothing here is approved. Anything adopted must move into
[[29 - Implementation Roadmap]] and, if it changes architecture, be recorded in
[[26 - Decisions]].

## Rule

> If it is not in this file, it does not get built.

---

## Deliberately out of scope

Recorded so they are not re-proposed each cycle.

| Idea | Why not |
|---|---|
| Real-money trading | Non-goal. Requires explicit authorization after extensive paper validation. |
| High-frequency market making | Wrong problem — see L-16. |
| Multi-user / auth / tenancy | Single desktop user. |
| Short selling / leverage | Leverage 1, spot-style. |
| Cloud deployment | Local-first. |
| Order-book replay we don't have | Cannot fabricate data ([[27 - Known Limitations]] L-08). |

---

## Candidate ideas

### Analysis

| Idea | Value | Prerequisite |
|---|---|---|
| Per-strategy and per-regime attribution | Separates edge from luck | Phase 6 |
| Slippage-vs-model error tracking | Calibrates the fill model | Phase 9 |
| Feature importance analysis | Prunes dead features | Enough data |
| Replay a past decision through the current system | Detects behaviour drift | Phase 9 |
| Correlation between Laya confidence and outcome | Direct lift measurement | Phase 10 |

### Data

| Idea | Value | Prerequisite |
|---|---|---|
| Locally cached historical candles | Removes repeated fetching | Phase 3 |
| Order-book snapshots archived over time | Builds the history we lack | Budget + storage |
| Multi-provider cross-check | Detects provider data errors | Two providers |

### Execution

| Idea | Value | Prerequisite |
|---|---|---|
| Depth-aware slippage | More realistic fills | Historical depth |
| Partial-fill queue modelling | Realistic limit fills | Level-3 data |
| Latency simulation | Honest backtests | Measured latency |

### Research

| Idea | Value | Prerequisite |
|---|---|---|
| Fine-tune Laya on our labelled decisions | Turns filter into signal | Labelled data |
| Walk-forward automated validation | Stronger evidence | Phase 9 |
| Online calibration monitoring | Detects drift | Phase 10 |

### Interface

| Idea | Value | Prerequisite |
|---|---|---|
| Candlestick + CVD chart | Better visual analysis | Vendored chart lib |
| Decision timeline view | "What was the AI doing at 14:32?" | Phase 12 |
| Journal browser with filters | Research ergonomics | Phase 12 |
| Backtest comparison view | Compare variants | Phase 12 |

### Architecture

| Idea | Value | Prerequisite |
|---|---|---|
| Postgres migration path | Only if scale demands it | Not currently needed |
| Second decision engine (e.g. Jev) | Redundancy | Interface already permits |
| Web dashboard (separate from chat) | Larger viewport | Phase 12 |
| MCP server for external agents | Extensibility | Phase 12 |

---

## Deliberately excluded

| Idea | Why |
|---|---|
| Self-modifying production strategy | Overfits. [[16 - Self Improvement]] |
| Automatic model promotion on new data | Same data, same result — not evidence. |
| GPU auto-scaling | Single machine. |
| Mobile app | Desktop tool. |

---

## Promotion criteria

An idea becomes roadmap scope when:

1. It has a stated purpose and a measurable success criterion.
2. It does not violate [[02 - Goals and Non Goals]].
3. It does not weaken [[20 - Security]] or the risk boundary.
4. Its cost (time, money, complexity) is documented in [[24 - Free Services]].
5. It is approved by the user.

Only then does it move to [[29 - Implementation Roadmap]].

---

[[02 - Goals and Non Goals]] · [[16 - Self Improvement]] · [[27 - Known Limitations]] · [[29 - Implementation Roadmap]]

## See also

- [[26 - Decisions]] — architectural changes
- [[25 - Research Log]] — evidence before promotion