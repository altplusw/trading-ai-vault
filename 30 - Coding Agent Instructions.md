---
tags: [agent, instructions, rules]
---

# 30 - Coding Agent Instructions

## Purpose

Binding rules for any coding agent working on this project — human-supervised or autonomous.

## The vault is the specification

**Chat history is not the specification. This vault is.**

Before implementing anything:

1. Read the relevant vault documents.
2. Inspect the existing repository.
3. Identify dependencies.
4. Make a **small** implementation plan.
5. Write tests.
6. Implement.
7. Run tests.
8. Update documentation **if architecture changed**.
9. Record important decisions.
10. Report exactly what changed — files, tests, and anything surprising.

Do not assume earlier conversation. Do not rely on undocumented requirements.

---

## The 20 rules

1. **Read the vault before implementing anything.**
2. **Treat the vault as the current specification.**
3. **Do not silently change architecture.** Architecture changes are deliberate, recorded,
   and reviewed.
4. **If implementation reveals the spec is wrong or impossible, document it and update the
   design document *before* changing the architecture.** Record it in [[27 - Known Limitations]]
   and, if it changes a decision, in [[26 - Decisions]].
5. **Never invent external APIs.** Inspect the real repository or documentation. If it cannot
   be verified, say so — do not guess. (See L-06 for a live example.)
6. **Do not rewrite working components** to suit a new style.
7. **Never disable a test to make a build pass.** Fix the code or document why.
8. **Never bypass the risk engine.** No exception, including "just for a backtest".
9. **Never enable real-money trading by default.** There is no live execution code.
10. **Never commit secrets.** `.env` is gitignored; `.env.example` is committed.
11. **Keep interfaces modular.** `LLMProvider`, `DecisionEngine`, `MarketDataProvider`,
    `Broker` stay replaceable.
12. **Keep the LLM provider replaceable.** Never hard-code a model name.
13. **Keep Laya replaceable** behind `DecisionEngine`. Only the adapter imports `laya`.
14. **Keep market data providers replaceable.** Only the adapter imports a provider SDK.
15. **Keep paper trading independent from live trading.** No shared code paths that could
    couple them.
16. **Every feature needs appropriate tests**, including its failure paths.
17. **Record architecture changes in [[26 - Decisions]].**
18. **Record unresolved problems in [[27 - Known Limitations]].**
19. **Record future ideas in [[28 - Future Ideas]]** rather than silently adding scope.
20. **Report exactly what changed** — files touched, tests run, results, and anything that
    surprised you.

---

## Non-negotiable safety invariants

These are not style preferences. A change that violates one is a defect regardless of test
results.

| Invariant | Where enforced |
|---|---|
| No order without a `risk_decision_id` | Broker signature |
| Risk limits come only from config | Risk engine signature |
| `REAL_TRADING=true` refuses to boot | `assert_paper_only()` |
| No live exchange SDK imported | `tests/test_safety.py` |
| Kill switch not clearable by any model/tool | `EmergencyStop` API |
| Any dependency failure ⇒ no trade | Risk engine fail-closed |
| Reasoning stored at decision time | Journal written in-transaction |
| No credential-shaped references in source | `tests/test_safety.py` |

---

## Honesty requirements

These are about reporting, and they matter as much as the code.

- **Never claim a feature works without running it.** State what was actually verified.
- **Never claim profitability.** Report measured metrics with sample size, assumptions, and
  whether the result is in-sample.
- **Never report a component as implemented** unless it exists and its tests pass.
- **Never fabricate an API, a number, or a citation.** If unverified, say "unverified".
- **When blocked, say blocked.** Do not substitute a plausible answer.
- **Report negative and null results** in [[25 - Research Log]] with the same care as positive
  ones.
- If a test was skipped, say so. A skipped test is not a passing test.

---

## Reporting format

For each change:

```
What changed:      <files and what they now do>
Tests:             <added / modified> — <result, with counts>
Verified:          <what you actually ran or executed>
Not verified:      <what you did not>
Spec changes:      <vault documents updated, if any>
Decisions:         <recorded in 26, if any>
Limitations found: <recorded in 27, if any>
Risks:             <anything a human should decide>
```

---

## Phase discipline

- Implement **one phase** at a time. Do not build ahead.
- Respect the gates in [[29 - Implementation Roadmap]]. Do not skip one.
- Do not progress past a gate without it passing.
- **Phase 14 (live trading) requires explicit human authorisation and is not planned.** It is
  never reached by proceeding automatically through phases.

---

## Working style

Prefer: simple architecture · small modules · clear interfaces · strong typing ·
deterministic core logic · testable functions · explicit error handling · structured logging ·
configuration over hard-coded values.

Avoid: giant files · hidden global state · hard-coded keys · provider-specific logic
scattered through the app · model-controlled risk limits · speculative abstraction · unused
dependencies · premature optimisation.

Match the style of the file you are editing. A diff full of formatting noise gets rejected.

---

[[00 - Home]] · [[26 - Decisions]] · [[27 - Known Limitations]] · [[29 - Implementation Roadmap]]

## See also

- [[21 - Testing]] — what each phase must demonstrate
- [[20 - Security]] — the invariants in full
- [[03 - System Architecture]] — the trust boundaries you must preserve