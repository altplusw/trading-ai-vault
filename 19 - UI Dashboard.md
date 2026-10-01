---
tags: [ui, dashboard]
---

# 19 - UI Dashboard

## Purpose

Make the architecture **visible**. The user should be able to see what the AI is doing, why,
and what it decided — not a black box that emits trades.

> The product should feel like *ChatGPT + Trading Terminal + Paper Exchange + Autonomous Agent*.

## Layout

```
┌──────────────────────────────────────────────────────────────────┐
│  Trading AI   [PAPER]   Space Bunny ●   Laya ●      Autonomous: off │
├───────────────────────────────┬──────────────────────────────────┤
│                               │  PAPER ACCOUNT                   │
│  CHAT                         │  balance · equity · available    │
│                               │  realised P&L · unrealised P&L   │
│  (natural language, left)     │  fees · exposure · daily P&L     │
│                               ├──────────────────────────────────┤
│                               │  POSITIONS                        │
│                               ├──────────────────────────────────┤
│                               │  ORDERS                           │
│                               ├──────────────────────────────────┤
│                               │  RECENT TRADES                    │
│                               ├──────────────────────────────────┤
│                               │  MARKET  price · 24h · volume    │
│                               │          chart · book · CVD      │
│                               ├──────────────────────────────────┤
│                               │  AI ACTIVITY                      │
│                               ├──────────────────────────────────┤
│                               │  AUTONOMOUS MODE                  │
└───────────────────────────────┴──────────────────────────────────┘
```

## Panels

**Chat** — natural language. Show which tools each turn used, with the ability to expand and
inspect raw results. Shows Laya's contribution distinctly from the Main AI's reasoning.

**Portfolio** — balance, equity, cash, realised, unrealised, fees, exposure. Realised and
unrealised **always separate**. Stale marks visibly labelled stale ([[11 - Portfolio and Positions]]).

**Positions** — symbol, side, qty, entry, mark, P&L, stop, target, strategy.

**Orders** — open / filled / cancelled / rejected, with rejection reasons visible.

**Market** — selected symbol, price, 24h change, volume, chart, order book, CVD, order-flow
panel, liquidation info where available.

**AI Activity** — request → tools selected → Laya decision + probability → proposal → risk
decision → execution → result. **This is the transparency panel and the most important one.**

**Autonomous Mode** — OFF / PAUSED / RUNNING / STOPPED / **EMERGENCY STOP**, visually
unmistakable.

## Design principles

1. **Nothing hidden.** If the AI used a tool, the user can see it and its result.
2. **Honest states.** Degraded service, stale data, and unavailable components are visible,
   not silently smoothed over.
3. **Paper mode always visible.** A persistent badge. Never ambiguous.
4. **Honest numbers.** Trade count next to any rate. Realised vs unrealised separated.
5. **Attribution.** Every figure traceable to its `decision_id`.
6. **Safe by construction.** No control that bypasses risk; no control that clears the kill
   switch except an explicit, unmistakable human action.

## Current state (Phase 1)

Implemented in `trading-ai/ui/index.html` — single file, no build step, no CDN (local-first):

- Header: mode badge, Space Bunny + Laya health dots, autonomous toggle
- Chat pane with tool chips (hover for raw JSON), Ctrl+Enter to send
- Paper account card, positions table, recent decisions table
- Build-status panel naming the current phase
- Polls `/api/health`, `/api/portfolio`, `/api/decisions`, `/api/autonomous` every 5 s

Not yet: chart, order book, CVD panel, AI activity timeline, journal view, backtest UI,
calibration view, settings editor.

## Charts

Any charting library must be vendored locally or served from our own host — **no CDN**, since
the project is local-first and must work offline. Evaluate options at Phase 12; a vendored
lightweight canvas library is preferable to a CDN dependency.

## Accessibility and honesty notes

- Never rely on colour alone for state (use text + icon).
- Negative P&L uses an explicit sign, not only red.
- Empty states say *why* they are empty ("no trades yet" vs "market data unavailable").

## Testing

- Panels render with an empty account without errors.
- Stale marks produce a visible stale indicator.
- Kill switch state is unmistakable in every state.
- Tool calls render with inspectable raw results.
- Paper badge is present on every page.
- Numbers reconcile with the database (spot-check against SQL).

---

[[11 - Portfolio and Positions]] · [[18 - API and Tools]] · [[12 - Trade Journal]] · [[22 - Deployment]]

## See also

- [[20 - Security]] — which controls the UI may expose
- [[29 - Implementation Roadmap]] — Phase 12
- `trading-ai/ui/index.html` — current implementation