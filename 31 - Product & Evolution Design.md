---
tags: [design, architecture, ai-ops, research, product]
---

# 31 - Product & Evolution Design

## Purpose

This document captures cross-cutting product requirements identified during architecture review:

- multiple LLM providers/models instead of one permanent model;
- free models may be rate-limited or disappear, so routing and fallbacks are required;
- context length is a model capability, not permanent memory;
- Laya must be updateable without silently changing the trading system;
- external analysts, news, reports, and videos may become research evidence, but never trading commands;
- the browser UI is the user's paper-trading control room.

This is a **design supplement**, not permission to skip roadmap gates.

## 1. AI Provider and Model Manager

### Goal

Allow the user to configure several providers and models and switch between them without changing trading-engine code.

API keys are user configuration/secrets. They must never be committed to the repository, logged, or stored in normal database rows.

### Responsibilities

| Capability | Required information |
|---|---|
| Provider | name, enabled/disabled, endpoint type |
| Model | model ID, provider, role |
| Context | maximum context length |
| Capabilities | tool calling, structured output, vision if applicable |
| Cost | input/output cost or free-tier status |
| Limits | requests/minute, requests/day, budget/day where known |
| Health | last successful call, last error, cooldown |
| Version | provider/model revision when available |

The manager exposes an `LLMProvider` interface. The rest of the application must not import provider-specific SDKs directly.

### Deterministic routing

The system should choose a model using explicit policy, not ask one LLM to randomly choose another LLM.

Example policy:

- simple extraction/classification → fast/cheap model;
- ordinary reasoning → configured general model;
- difficult reasoning → stronger model if allowed by budget;
- very large research context → model with sufficient context;
- provider/model unavailable → configured fallback;
- free-only mode → reject paid candidates.

### Modes

The UI should eventually expose:

- **Free Only** — never spend money;
- **Balanced** — prefer free/local, allow explicitly configured paid fallback;
- **Quality** — allow paid models within a user-set budget.

Every remote call should record provider, model, latency, success/failure, tokens when available, and estimated cost when available.

## 2. Context Manager

Context length is a per-request capacity, not permanent memory.

The system should build task-specific context instead of dumping the entire database into every prompt.

Potential inputs:

- current market state;
- recent candles/order-flow data;
- current portfolio and positions;
- risk configuration;
- recent relevant trades;
- relevant research evidence;
- recent conversation;
- selected tool results.

The Context Manager should select and order only relevant information and enforce the target model's context limit.

### Important distinction

**Context** = information sent to the model for one request.

**Memory** = information persisted in the database and retrieved later.

A large-context model does not automatically remember everything forever.

### Context failure policy

If the selected model cannot safely fit the required context:

1. retrieve less information using relevance rules;
2. summarize only where the summary is explicitly acceptable;
3. route to a model with a larger context window; or
4. fail closed for tasks where loss of context would make the decision unsafe.

Never silently truncate decision-critical market or risk data.

## 3. Laya Update Manager

Laya is an external dependency and should not be permanently frozen to an old version.

However, automatic replacement is unsafe.

### Update lifecycle

```
Upstream release
      ↓
Update detector
      ↓
Read release notes / source changes
      ↓
Create isolated update branch
      ↓
Update Laya adapter/dependency/checkpoint if required
      ↓
Run compatibility tests
      ↓
Run Laya latency benchmark
      ↓
Run decision benchmark / regression suite
      ↓
Compare old vs new
      ↓
Human approval
      ↓
Merge
```

The coding agent may prepare the update, but it must not silently replace the production/default Laya version.

Every accepted Laya update records:

- upstream version/commit;
- local version;
- dependency/checkpoint changes;
- benchmark results;
- regression results;
- known limitations;
- date of approval.

The project should be able to roll back to the previous known-good version.

## 4. External Research and Evidence

### Goal

Allow the system to consume research from external analysts, financial reports, news, transcripts, and similar sources.

The system must preserve attribution and uncertainty.

An external claim is **evidence**, not an instruction.

Example record:

| Field | Example |
|---|---|
| Source | analyst/video/report |
| Author | named source |
| Published | timestamp |
| Asset/pair | EUR/USD |
| Directional claim | EUR may weaken |
| Time horizon | 1–3 months |
| Reasons | source's stated reasons |
| Source URL | original source |
| Extraction status | verified / uncertain |
| Outcome | measured later |

The Main AI may consider research evidence together with market data, but the research system must never directly execute a trade.

### Outcome tracking

Predictions/claims should be retained so the system can later compare:

- what was claimed;
- when it was claimed;
- the stated horizon;
- what actually happened;
- relevant market conditions.

Historical performance of a source is evidence about that source's past record, not proof that its next prediction is correct.

## 5. Paper-Trading Dashboard

The browser UI is a first-class product surface.

It should feel like:

**AI assistant + trading terminal + paper exchange + autonomous agent.**

### Persistent top-level state

Always show:

- PAPER mode badge;
- market-data status;
- Laya status;
- Main AI status;
- Risk Engine status;
- autonomous state;
- emergency-stop state.

### Paper account

At startup/configuration, allow the user to choose an initial paper balance, including a custom amount such as $100, $1,000, or $10,000.

Track at minimum:

- starting balance;
- current cash;
- equity;
- realised P&L;
- unrealised P&L;
- fees;
- return %;
- exposure;
- drawdown.

### Visual market panel

The dashboard should include:

- live/current price;
- candle or line chart;
- volume;
- buy markers;
- sell markers;
- current position;
- entry price;
- P&L;
- order-flow panels when data exists.

Charts must make it visually obvious where simulated trades occurred.

### AI activity panel

Show the decision pipeline:

```
Market data
    ↓
Laya
    ↓
Main AI
    ↓
Trade proposal
    ↓
Risk Engine
    ↓
Paper Broker
    ↓
Result
```

A user should be able to inspect:

- which model/provider was used;
- Laya decision and probabilities;
- relevant tools/data;
- proposal;
- risk verdict;
- simulated execution;
- later result.

### Controls

The dashboard should provide clear human controls for:

- start paper trading;
- pause;
- stop;
- emergency stop;
- reset paper account;
- choose market/symbol;
- choose model/provider profile;
- choose initial paper balance.

The UI must never imply that paper execution is real-money execution.

## 6. Design rule: live information vs live money

The project can consume **current/live market data** while still remaining **paper-only**.

The intended development loop is:

```
REAL CURRENT MARKET DATA
        ↓
        AI
        ↓
DETERMINISTIC RISK ENGINE
        ↓
PAPER BROKER
        ↓
FAKE / SIMULATED ACCOUNT
```

Real-money execution is outside the scope of the current project.

## 7. Design consequences for existing phases

The roadmap should incorporate these as cross-cutting requirements:

- Phase 1: provider/model configuration and secret handling foundations.
- Phase 2: model routing, fallback, usage accounting, and context management.
- Phase 5: Laya version pinning, update testing, and benchmark recording.
- Phase 8: store model/provider/version and relevant research evidence with trade decisions.
- Phase 11: enforce request/cost/escalation budgets and dependency fail-closed behavior.
- Phase 12: deliver the visual paper-trading control room and settings.
- Phase 13: research-source outcome tracking may inform experiments, but nothing auto-deploys.

## 8. Non-goals

This document does not authorize:

- autonomous real-money trading;
- blindly auto-upgrading Laya;
- blindly trusting a public commentator;
- spending money without a configured budget;
- sending unrestricted database history to an LLM;
- letting an LLM change risk limits;
- letting an LLM approve its own model/dependency updates.