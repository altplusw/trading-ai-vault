---
tags: [database, persistence]
---

# 17 - Database

## Purpose

Provide **memory**: the audit trail, the trading history, and the training corpus. Design
for comprehensibility — a human must be able to answer "why did this trade happen?" with a
query.

## Choice: SQLite

**SQLite** with WAL mode and foreign keys enabled. Rationale:

| Need | SQLite |
|---|---|
| Single-user local desktop | ✅ ideal |
| Zero administration | ✅ file only |
| Relational integrity (positions ↔ orders ↔ trades) | ✅ real FKs |
| Auditable queries | ✅ plain SQL |
| Concurrent readers + one writer | ✅ WAL |
| Portability | ✅ one file + JSON export |
| Server deployment | ⚠️ fine at this scale; migration path to Postgres exists |

Rejected: Postgres (operational burden for one user), Mongo (relational integrity is the
point), in-memory (data loss). Migration seam: standard SQL, no ORM lock-in.

## Conventions

- **Timestamps: REAL unix epoch seconds.** Ordering, indexing, and duration arithmetic work
  directly. Convert at the edge.
- **JSON payloads: TEXT**, opaque to the schema, owned by the application.
- **Money: TEXT or INTEGER minor units**, never float. Python side uses `Decimal`.
- **Every consequential row carries `decision_id`.**
- **`source`/`created_by`** on orders and decisions: `USER` · `AUTONOMOUS_AI` · `BACKTEST` · `MANUAL`.
- **Version columns** on strategies, models, question batteries, and config, so history
  stays attributable after things change.
- **Nothing is deleted.** Retention applies to raw API responses only.

## Schema

Authoritative DDL: `trading-ai/database/schema.sql`. Tables:

| Table | Purpose |
|---|---|
| `conversations`, `messages` | chat history, including tool calls made |
| `market_snapshots` | what the market looked like at decision time |
| `laya_decisions` | raw Laya output per evaluation |
| `ai_decisions` | model reasoning + proposal + risk verdict + final action |
| `trade_proposals` | structured proposals |
| `risk_decisions` | every check, passed and failed, with reasons |
| `orders`, `fills` | order lifecycle and executions |
| `positions` | current net position per symbol |
| `accounts`, `ledger` | balances + append-only money ledger |
| `trades` | round trips with P&L |
| `trade_journal` | human-readable narrative per decision |
| `equity_points` | equity curve series |
| `calibration_records` | prediction + probability + outcome |
| `strategies`, `strategy_versions` | immutable versioned strategies |
| `backtest_runs` | every run incl. failures |
| `models`, `model_versions` | Laya/LLM versions used per decision |
| `research_proposals` | candidates from the research loop |
| `system_events`, `audit_events` | operational and security audit |

Implemented so far: `conversations`, `messages`, `market_snapshots`, `ai_decisions`,
`orders`, `positions`, `accounts`, `ledger`, `trades`, `trade_journal`. The rest are
specified here and added per phase.

## Constraints

Enforced by the schema, not just application code:

- `CHECK` on every enum (`source`, `side`, `order_type`, `status`, `proposed_action`,
  `risk_result`, `qtype`-equivalent, `role`, `kind`, `result`).
- `CHECK (quantity > 0)`, `CHECK (filled_quantity >= 0)`.
- Foreign keys with deliberate `ON DELETE` behaviour — mostly `RESTRICT` or `SET NULL` so
  history is never silently orphaned.
- `UNIQUE` where duplicates would be a bug (e.g. idempotency keys).

Tests assert invalid values raise, not silently coerce ([[21 - Testing]]).

## Write discipline

- **One transaction per decision.** Journal entry, market snapshot, decision, risk
  verdict, and order writes commit together or not at all. A partially recorded decision is
  worse than none.
- Idempotency keys on autonomous cycles to prevent duplicate orders on retry.
- WAL + `busy_timeout`. One writer at a time; short transactions.

## Query patterns the schema must support

```sql
-- Why did this trade happen?
SELECT * FROM ai_decisions  WHERE decision_id = ?;
SELECT * FROM trade_journal WHERE decision_id = ?;
SELECT * FROM laya_decisions WHERE decision_id = ?;
SELECT * FROM risk_decisions WHERE decision_id = ?;
SELECT * FROM orders         WHERE decision_id = ?;
SELECT * FROM trades         WHERE decision_id = ?;

-- Has any Laya question shown predictive value?
SELECT question_id, COUNT(*), AVG(probability), AVG(outcome)
  FROM calibration_records GROUP BY question_id;

-- Which risk rule blocks most?
SELECT rule, COUNT(*) FROM risk_decisions, json_each(reasons)
 WHERE NOT approved GROUP BY rule ORDER BY 2 DESC;
```

These are the queries the product promises. If they are slow or impossible, the schema is
wrong.

## Migration

Simple ordered SQL files, applied once, recorded in `schema_migrations`. No ORM, no
auto-generation. Keep the DDL in git as the source of truth.

## Backup and export

- Single file ⇒ `VACUUM INTO` for a consistent snapshot.
- JSON export of any decision by `decision_id` for external analysis.
- The DB **is** the research corpus — export to Parquet for the training pipeline
  ([[16 - Self Improvement]]).

## Security

- File permissions restricted; the DB may contain strategy and performance data.
- **No secrets stored.** API keys live in `.env` only ([[20 - Security]]).
- Never store raw credential material in any table.

## Testing

- Schema applies cleanly to an empty DB; re-applying is idempotent.
- Every CHECK rejects its invalid value.
- FK enforcement (attempt an orphan insert → raises).
- Transaction atomicity: injected failure mid-decision leaves no partial rows.
- Idempotency: replaying a cycle inserts nothing new.
- Documented query patterns return complete, correct result sets.
- `VACUUM INTO` produces a loadable copy.

---

[[12 - Trade Journal]] · [[21 - Testing]] · [[23 - Configuration]] · [[14 - Calibration]]

## See also

- `trading-ai/database/schema.sql` — the implemented DDL
- [[10 - Paper Trading]] — which rows fills produce
- [[09 - Risk Engine]] — risk_decisions table requirements