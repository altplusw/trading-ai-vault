---
tags: [security, safety, critical]
---

# 20 - Security

## Purpose

Prevent loss of funds, credential leakage, and unintended real-money execution. Treat every
other requirement as subordinate to this document.

## Threat model

| Threat | Control |
|---|---|
| A model places a real order | No live execution code exists. Verified by test. |
| A model overrides a risk limit | Risk engine takes limits only from config ([[09 - Risk Engine]]) |
| A model disables the kill switch | No tool clears it; human-only |
| Secrets committed to git | `.env` gitignored; `.env.example` committed; CI greps |
| Secrets logged | Redaction filter; no credential-shaped logging |
| Withdrawal-capable credentials | Never requested, never stored |
| Loop runs away with funds | Rate limits, order frequency caps, daily loss limit, cooldown |
| Stale data causes a bad trade | Freshness check in risk engine; fail closed |
| Local service exposed to network | Binds `127.0.0.1`; auth if configured |
| Untrusted input crashes the system | Validated inputs; explicit error handling |

## Hard rules

### 1. No real-money execution

There is **no exchange SDK installed or imported**, and no code path that could place a live
order. Enforced and tested:

- `tests/test_safety.py` asserts no exchange SDK is importable.
- A test greps all source for credential-shaped env vars and fails on any hit.
- `assert_paper_only()` **raises** if `REAL_TRADING=true` — it does not branch. A branch
  would mean a `true` value silently re-enables a path that does not exist; a raise makes
  flipping it loud, deliberate, and reviewable.

Live trading, if ever, is a **new broker implementation** in a separate package, requiring
explicit human authorization ([[29 - Implementation Roadmap]] Phase 14).

### 2. Secrets

- `.env` — never committed, gitignored, created from `.env.example`.
- API keys, private keys, passwords, 2FA codes, wallet seeds — **never** in code, DB,
  logs, or documentation.
- Keys read from environment only, at the point of use.
- Withdrawal permissions must **never** be required of any key.
- Never store raw credentials in any database table.

### 3. Network exposure

- Default bind `127.0.0.1`. **Not** `0.0.0.0`.
- If an API key is configured, require bearer auth, compared in constant time on **bytes**
  (Starlette decodes headers as latin-1; `compare_digest` raises on non-ASCII `str`).
- No CORS wildcard.
- Internet access is outbound only, to market-data and LLM providers.

### 4. Input validation

The model is an untrusted input source. Every tool validates:

- Symbol against the whitelist.
- Quantity/price positive, finite, within limits.
- Timestamps sane; reject impossible values.
- JSON depth and size bounded.
- No shell, no eval, no dynamic imports, no SQL string interpolation — **parameterised
  queries only**.

### 5. Resource limits

Bound everything that could be used to exhaust the machine: request size, question count,
option count, concurrency, loop iterations, tool-call rounds, escalation budget, order
frequency.

### 6. Audit

Log security-relevant events: config changes, kill-switch changes, mode changes, auth
failures, risk rejections, all orders, strategy version changes, DB migrations.

## Implementation status

| Control | Status |
|---|---|
| Paper-only enforced at boot | ✅ implemented, tested |
| `REAL_TRADING` raises | ✅ implemented, tested |
| No exchange SDK importable | ✅ tested |
| No credential-shaped source references | ✅ tested |
| Secrets in `.env` only | ✅ `.gitignore` + `.env.example` |
| Localhost-only bind | ✅ |
| Structured logging with redaction | ⚠️ logging implemented; **redaction not yet** |
| API key auth | ⬜ Phase 2 |
| Request/concurrency limits | ⬜ Phase 2 |
| Parameterised queries everywhere | ✅ (SQLite, parameterised) |
| Audit events | ⬜ Phase 7 |

⚠️ **Gaps are real gaps.** Redaction and request limits are not yet implemented and are
listed as open in [[27 - Known Limitations]].

## Secret handling checklist for any future change

Before adding a provider or broker:

- [ ] Does it need a key with withdrawal permission? **It must not.**
- [ ] Is the key read from environment only?
- [ ] Is it excluded from logs, including exception text?
- [ ] Is it excluded from the database?
- [ ] Does it fail **closed** when absent?
- [ ] Is the network exposure unchanged (outbound only)?

## Testing

- `REAL_TRADING=true` ⇒ startup refuses.
- No exchange SDK importable.
- No credential-shaped strings in source.
- Kill switch engaged ⇒ all order types rejected; not clearable via tool.
- Broker rejects any submission without `risk_decision_id`.
- Invalid tool arguments rejected, never coerced.
- SQL injection attempt in a tool argument is parameterised away.
- Oversized request rejected before processing.
- Auth: missing/invalid key ⇒ 401.
- Logs contain no secret values.

---

[[09 - Risk Engine]] · [[23 - Configuration]] · [[21 - Testing]] · [[22 - Deployment]]

## See also

- [[02 - Goals and Non Goals]] — real trading is a non-goal
- [[26 - Decisions]] — why the raise-not-branch pattern
- [[15 - Autonomous Trading]] — loop-level limits