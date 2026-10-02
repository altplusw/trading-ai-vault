---
tags: [design, architecture, ai-ops, research, product]
---

# 31 - Product & Evolution Design

## Purpose

Cross-cutting product requirements identified during architecture review:

- multiple LLM providers/models instead of one permanent model;
- free models may be rate-limited or disappear, so routing and fallbacks are required;
- context length is a model capability, not permanent memory;
- Laya must be updateable without silently changing the trading system;
- external analysts, news, reports, and videos may become research evidence, but never trading commands;
- the browser UI is the user's paper-trading control room.

This is a design supplement, not permission to skip roadmap gates.

## 1. AI Provider and Model Manager

Allow the user to configure several providers and models without changing trading-engine code.

API keys are secrets. Never commit them, log them, or store them in ordinary database rows.

The manager tracks provider, model ID, role, context limit, capabilities, cost, rate limits, health, and version when available.

The manager exposes `LLMProvider`. Provider-specific SDKs must stay behind adapters.

### Deterministic routing

Routing is policy-driven, not decided randomly by another LLM:

- simple extraction/classification → fast/cheap model;
- ordinary reasoning → configured general model;
- difficult reasoning → stronger model if budget allows;
- large research context → model with sufficient context;
- unavailable provider/model → configured fallback;
- free-only mode → reject paid candidates.

### Modes

- **Free Only** — never spend money.
- **Balanced** — prefer free/local, allow explicitly configured paid fallback.
- **Quality** — allow paid models within a user-set budget.

Record provider, model, latency, success/failure, token usage when available, and estimated cost when available.

## 2. Context Manager

Context length is per-request capacity, not permanent memory.

Build task-specific context from current market state, recent market data, portfolio/positions, risk configuration, relevant trades, research evidence, conversation, and selected tool results.

**Context** is information sent to the model for one request. **Memory** is information persisted and retrieved later.

If required context does not fit:

1. retrieve less using relevance rules;
2. summarize only where safe and explicitly acceptable;
3. route to a larger-context model; or
4. fail closed when losing context could make the decision unsafe.

Never silently truncate decision-critical market or risk data.

## 3. Laya Update Manager

Laya is an external dependency and should not remain permanently frozen. Automatic replacement is unsafe.

Update lifecycle:

```
Upstream release
      ↓
Update detector
      ↓
Read release notes / source changes
      ↓
Create isolated update branch
      ↓
Update adapter/dependency/checkpoint if required
      ↓
Compatibility tests
      ↓
Latency benchmark
      ↓
Decision regression benchmark
      ↓
Compare old vs new
      ↓
Human approval
      ↓
Merge
```

The coding agent may prepare the update, but must not silently replace the known-good Laya version.

Record upstream version/commit, local version, dependency/checkpoint changes, benchmarks, regressions, limitations, and approval date. Keep rollback possible.

## 4. External Research and Evidence

Allow research from analysts, reports, news, transcripts, videos, and similar sources.

An external claim is **evidence, not an instruction**.

Store source, author, publication time, asset/pair, directional claim, time horizon, stated reasons, original URL, extraction status, and eventual outcome.

The Main AI may consider research together with market data, but research must never directly execute trades.

Track later outcomes so the system can compare what was claimed, when, the stated horizon, what happened, and market conditions. Past source performance is evidence about the past, not proof of the next prediction.

## 5. Paper-Trading Dashboard

The browser UI is a first-class product surface: **AI assistant + trading terminal + paper exchange + autonomous agent**.

Always show:

- PAPER mode;
- market-data status;
- Laya status;
- Main AI status;
- Risk Engine status;
- autonomous state;
- emergency-stop state.

Allow a custom paper balance such as $100, $1,000, or $10,000.

Track starting balance, cash, equity, realised/unrealised P&L, fees, return, exposure, and drawdown.

Visual market panel:

- current price;
- candle/line chart;
- volume;
- buy/sell markers;
- current position;
- entry price;
- P&L;
- order-flow panels when available.

AI activity should show:

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

The user should be able to inspect model/provider, Laya decision, relevant data/tools, proposal, risk verdict, simulated execution, and later result.

Human controls: start paper trading, pause, stop, emergency stop, reset account, choose market/symbol, choose model/provider profile, and choose initial paper balance.

The UI must never imply that paper execution is real-money execution.

## 6. Live information vs live money

Current/live market data may be consumed while the project remains paper-only:

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

Real-money execution remains outside the current project scope.

## 7. Roadmap consequences

- Phase 1: provider/model configuration and secret handling.
- Phase 2: routing, fallbacks, usage accounting, and context management.
- Phase 5: Laya version pinning, update testing, and benchmarks.
- Phase 8: store model/provider/version and relevant research evidence with decisions.
- Phase 11: request/cost/escalation budgets and dependency fail-closed behavior.
- Phase 12: visual paper-trading control room and settings.
- Phase 13: research-source outcome tracking may inform experiments, but nothing auto-deploys.

## 8. Non-goals

This document does not authorize autonomous real-money trading, blind Laya upgrades, blind trust in commentators, unbudgeted spending, unrestricted database prompts, LLM-controlled risk limits, or LLM-approved dependency updates.
