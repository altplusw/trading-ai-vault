---
tags: [llm, core]
---

# 04 - Main AI

## Purpose

The conversational and reasoning brain. It talks to the user, chooses tools, interprets
market data and Laya output, and **proposes** actions. It never executes.

## Provider abstraction

The main model must be swappable without touching the trading engine.

```python
class LLMProvider(Protocol):
    def chat(self, messages, tools=None, **kw) -> ModelTurn: ...
    def health(self) -> ProviderHealth: ...
    @property
    def model_id(self) -> str: ...
```

Implementations:

| Class | For |
|---|---|
| `OpenAICompatibleProvider` | Anything speaking the OpenAI chat-completions API: LM Studio, DeepSeek, OpenAI, most local servers |
| `MockProvider` | Deterministic replies for tests |

**Never hard-code a model name.** `LLM_MODEL` blank means auto-discover from
`GET {LLM_BASE_URL}/models`. `model_id` is recorded on every decision so history stays
attributable when the model changes.

### Configuration

```
LLM_PROVIDER=openai_compatible
LLM_BASE_URL=http://localhost:1234/v1
LLM_API_KEY=lm-studio
LLM_MODEL=
LLM_TEMPERATURE=0.4
LLM_MAX_TOKENS=1200
LLM_TIMEOUT_SECONDS=120
```

Verified working in `trading-ai/llm/client.py`: auto-discovery, `LLMUnavailable` on any
transport failure, own retry policy so failures stay visible.

## Responsibilities

- Natural conversation.
- Deciding **which** tools to call, and in what order.
- Interpreting market state and Laya output.
- Forming a `TradeProposal`.
- Explaining past trades from stored journal evidence.
- Proposing strategy/hypothesis changes (never applying them — [[16 - Self Improvement]]).

## Non-responsibilities — hard rules

- **Must not** answer current prices, balances, positions or P&L from memory. No tool, no claim.
- **Must not** place, modify or cancel an order.
- **Must not** widen a risk limit, override a rejection, or disable the kill switch.
- **Must not** describe a proposed trade as executed.
- **Must not** assert profitability.

These are enforced in the system prompt **and** structurally: the model has no tool that
places an order, and every order path runs the risk engine ([[09 - Risk Engine]]).

## Tool calling

Loop: model → tool calls → results → model → … until no tool calls or a round cap.

Each tool returns JSON. Tools return *errors as data* (`{"ok": false, "error": ...}`) rather
than raising, so one failing tool does not abort a turn.

Full tool list and schemas: [[18 - API and Tools]].

## Fail-closed behaviour

`LLMUnavailable` → no autonomous trade, and the chat replies that the model is unreachable.
Verified: with the LLM down, the system records zero decisions.

## Prompting notes

System prompt must:

1. State paper-only explicitly.
2. Forbid memory answers about live state.
3. Require separating observed data / Laya output / interpretation / risk verdict.
4. Forbid blind agreement with user assertions.
5. Forbid profit claims.
6. Explain that Laya output is a signal, not a certainty.

Implemented in `trading-ai/app/chat.py::SYSTEM_PROMPT`.

## Costs and latency

The Main AI is the expensive, slow component. That is why [[05 - Laya]] exists: filter
cheaply before invoking it. If Main AI is remote, note per-call cost and latency in
[[24 - Free Services]]; never let a loop call it unboundedly.

## Testing

- Provider returns each tool-call shape → loop dispatches correctly.
- Provider raises → `LLMUnavailable` → no decision recorded.
- Model returns malformed args → tool rejects, no execution.
- Round cap enforced → loop terminates.

---

[[03 - System Architecture]] · [[18 - API and Tools]] · [[24 - Free Services]] · [[05 - Laya]]

## See also

- [[15 - Autonomous Trading]] — when the Main AI is invoked automatically
- [[12 - Trade Journal]] — how past reasoning is stored