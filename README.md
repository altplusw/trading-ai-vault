# Trading AI — Project Vault

This Obsidian vault is the **permanent source of truth** for this project.

Chat history is not the specification. These documents are.

## What this is

A local-first AI trading chatbot that operates a **paper-trading** account. Real-money
trading does not exist in this codebase.

## Where things are

| Path | Contents |
|---|---|
| **Repository root** | Obsidian vault documentation and project specification |
| `trading-ai/` | The implementation (Python, FastAPI, SQLite) |

## Visual maps

| Canvas | What it shows |
|---|---|
| **[Trading AI - System Map.canvas](Trading%20AI%20-%20System%20Map.canvas)** | The whole architecture as clickable cards — model layer, trust boundary, memory, evidence |
| **[Trading AI - Roadmap Map.canvas](Trading%20AI%20-%20Roadmap%20Map.canvas)** | Build order, the 10 gates, and the open questions blocking each wave |

Most notes also contain inline [Mermaid](https://mermaid.js.org/) diagrams, which Obsidian
renders natively.

## Start here

- [[00 - Home]] — five-minute overview, current status, safety model
- [[29 - Implementation Roadmap]] — the 15 phases and approval gates
- [[26 - Decisions]] — architecture decisions and why
- [[27 - Known Limitations]] — what does not work yet, and open questions
- [[30 - Coding Agent Instructions]] — rules any agent must follow
- [[31 - Product & Evolution Design]] — cross-cutting product, AI-provider, Laya-update, research, and dashboard design

## Document index

**Foundations** — [[01 - Project Overview]] · [[02 - Goals and Non Goals]] · [[03 - System Architecture]]

**The two brains** — [[04 - Main AI]] · [[05 - Laya]] · [[06 - Market Data]] · [[07 - Order Flow]]

**Trading** — [[08 - Strategy Engine]] · [[09 - Risk Engine]] · [[10 - Paper Trading]] · [[11 - Portfolio and Positions]] · [[12 - Trade Journal]]

**Evidence** — [[13 - Backtesting]] · [[14 - Calibration]] · [[15 - Autonomous Trading]] · [[16 - Self Improvement]]

**Platform** — [[17 - Database]] · [[18 - API and Tools]] · [[19 - UI Dashboard]] · [[20 - Security]] · [[21 - Testing]] · [[22 - Deployment]] · [[23 - Configuration]] · [[24 - Free Services]]

**Meta** — [[25 - Research Log]] · [[26 - Decisions]] · [[27 - Known Limitations]] · [[28 - Future Ideas]] · [[29 - Implementation Roadmap]] · [[30 - Coding Agent Instructions]] · [[31 - Product & Evolution Design]]

---

**Status:** Phase 0 — Documentation
**Next:** Architecture review (Gate 2)