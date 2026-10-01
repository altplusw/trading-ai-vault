---
tags: [laya, decision-engine, core]
---

# 05 - Laya

> **Verification note.** Every API detail below was read from the source at
> [`github.com/NandhaKishorM/laya`](https://github.com/NandhaKishorM/laya), pinned at commit
> **`6d942c9` (v0.3.22)**, cloned to `../laya-repo`. Upstream `main` had moved **174 commits
> ahead** of this pin at the time of writing. Re-verify before Phase 5.

## Purpose

A **fast, local, non-autoregressive decision engine**. It answers typed questions about a
piece of state and returns probabilities. It does not generate text and does not trade.

## What it actually is (verified)

A prompt-conditioned **discriminative classifier** — PET-style pattern-exploiting
classification on a ModernBERT backbone. It is not an LLM and not a chatbot.

The input is serialized to a single token sequence (`laya/common.py::build_sequence`):

```
[CLS] <type> question: <instructions> [SEP] [MASK] opt0 [MASK] opt1 ... [SEP] <state> [SEP]
```

A bidirectional encoder runs **once**. Hidden states are gathered at the `[MASK]`
positions, and one shared MLP (`scorer`) projects each to a single logit. Softmax over
options gives the distribution.

Consequences that matter to us:

- **One forward pass per request**, no autoregressive decoding. Genuinely fast.
- **Options share one token budget.** The number of options you can express is limited —
  see *Limits* below.
- **The state is text.** `serialize_state()` accepts `str`, `dict`, or `list`, and JSON-dumps
  anything else. There is **no image, array, or binary input path.**

## Question types — exact semantics

Verified from `laya/agent.py`.

| Type | Input | Output | Notes |
|---|---|---|---|
| `choice` | `criteria: {label: description}` | `choice` (argmax), `probabilities` per label | A bare list of labels is accepted |
| `score` | `criteria: [level0, level1, …]` (ordered) | `score` = **probability-weighted expectation** | ⚠️ See below |
| `noul` | optional `criteria`/`labels` | `noul` = P(true) | Always exactly 2 options, always `[false, true]` order |

### ⚠️ `score` is an expectation, not a category

`score: 1.6` means "between level 1 and level 2, weighted by probability." It is **not**
"level 1." Getting a hard label requires argmaxing `probabilities` yourself.

`trading-ai/laya/client.py::summarise()` already does this for `setup_quality`. Any new
score question must do the same or it will be silently wrong.

### Confidence: two different numbers

| Field | Definition | Use |
|---|---|---|
| `answer_confidence` | `max(p)` — probability of the reported answer | **This is the calibrated one. Gate on this.** |
| `confidence` | `1 − H(p)/log(k)` — distribution concentration | Concentration, *not* calibration |

Temperature scaling fits `answer_confidence`. The `confidence` field is not calibrated and
shift with option count. **Never port a threshold from another system into `confidence`.**

## Installation and runtime

```bash
pip install "laya[serve]"          # adds FastAPI + uvicorn
LAYA_DEVICE=cpu LAYA_PRELOAD=1 laya-serve    # binds 0.0.0.0:8000
```

Checkpoint weights download from Hugging Face on first load (`convaiinnovations/laya`).
Requires Python ≥ 3.10, torch, transformers. `laya-serve` is **its own process** — it is
**not** a GGUF model and must **not** be loaded into LM Studio.

Existing implementation: `trading-ai/laya/client.py` — thin HTTP client, fail-closed.

## HTTP API (verified from `laya/serve.py`)

### `POST /v1/systemone`

Request body:

```json
{
  "state": { "body": "..." },
  "questions": {
    "dept": {"type": "choice", "instructions": "which team?",
             "criteria": {"billing": "refunds", "tech": "bugs"}}
  },
  "model": null, "task": null, "lang": null, "lang_guess": null,
  "min_confidence": null, "max_len": null, "head_max_len": null
}
```

Forwarded controls (verbatim from source):

```
BODY_CONTROLS = ("model", "max_len", "head_max_len", "task",
                 "lang", "lang_guess", "min_confidence")
```

**Refused with 422** (these are callables that would run in the server process):

```
BODY_REFUSALS = ("hooks", "on_predict_start", "on_predict_end",
                 "hooks_raise", "hooks_timeout")
```

Response:

```json
{
  "answers": {
    "dept": {"type": "choice", "choice": "billing",
             "probabilities": {"billing": 0.94, "tech": 0.06},
             "confidence": 0.81, "answer_confidence": 0.94,
             "action": {"act_probability": 0.5}}
  },
  "routing": {"model": "english", "repo": "...", "reason": "English Latin text"},
  "usage": {"input_tokens": 128, "output_tokens": 0}
}
```

### `POST /v1/systemone/batch`

`{"states": [...], "questions": {...}}` → `{"results": [...], "total_usage": {...}}`.

### `GET /health`

Reports loaded checkpoints, revisions, actual device, CPU-fallback counts.

### Server-side limits (hard-coded in `laya/serve.py`)

| Limit | Value | Meaning |
|---|---|---|
| `MAX_QUESTIONS` | 64 | questions per request |
| `MAX_STATE_CHARS` | 50000 | state size |
| `MAX_BATCH_STATES` | 64 | states per batch call |
| `MAX_BODY_BYTES` | 2 MiB | request size |
| `MAX_CHOICE_OPTIONS` | 100 | options per choice question → **413** |
| `MAX_SCORE_LEVELS` | 32 | levels per score question |
| `MAX_TOTAL_OPTIONS` | 512 | summed across the request |
| `LAYA_MAX_CONCURRENT` | 16 | in-flight requests |
| `DEFAULT_MAX_TOKEN_BUDGET` | 8192 | caps `max_len` override |

Exceeding these is **rejected before inference**, not silently truncated.

## Hard limits — read before designing questions

### 1. No vision. Text only.

`serialize_state` is `str` or `json.dumps`. There is no PIL/cv2/image path. **Laya cannot read
a chart or an image.** Any visual analysis needs a different tool.

### 2. Token budget is small

| Checkpoint | Context | `head_max_len` (option budget) |
|---|---|---|
| `laya` (English, ModernBERT-large 421M) | 512 | 192 |
| `laya-multilingual` (mmBERT-base 322M) | 1024 (up to 8192 w/ RoPE) | 256 |
| `laya-typed-decisions` (ModernBERT-large) | 1024 | 256 |

The real state budget is `max_len − head_len − 1` where `head_len` is what was *actually*
built — **not** `max_len − head_max_len`. Log and check `usage` for truncation.

### 3. Many options break it

Options share one budget. Past roughly `head_max_len / 4` options the head overflows the cap
and options become indistinguishable. Measured by the project: **Banking77 scores 0.425
across 77 options.** Keep questions to roughly **≤ 20 options**; beyond that, narrow with
`predict_shortlist` or split the question.

### 4. Zero-shot accuracy on domain tasks is near chance

On the project's own typed-decisions benchmark the **shipped** checkpoints score **0.362 /
0.352** against a **0.461 majority-class baseline**. The headline 0.766 comes from a
fine-tuned checkpoint (~2 GPU-hours).

**Therefore: treat Laya as a cost filter first and a signal second.** Even an untuned Laya
saves money by filtering events before they reach the expensive Main AI. Signal requires
fine-tuning on labelled market data ([[14 - Calibration]], Phase 10).

### 5. It fails confidently on wrong domains

The English checkpoint scores **0.000 on Khmer while reporting 95.2% confidence.** Confidence
gating cannot save you from a model that is confidently wrong. Verify on our own data.

### 6. Checkpoint marker file

A Laya checkpoint directory must contain `rl_agent_config.json`. Its absence is the error
meaning "this is not a Laya model." Useful for validating local model directories.

## DecisionEngine abstraction

Laya must be swappable. The application must not import `laya` outside the adapter.

```python
class DecisionEngine(Protocol):
    def evaluate(self, state: MarketState, battery: QuestionBattery) -> DecisionResult: ...
    def health(self) -> EngineHealth: ...
    @property
    def version(self) -> str: ...   # recorded on every decision
```

Implementations: `LayaDecisionEngine`, `MockDecisionEngine`, and later
`JevDecisionEngine` if a Jev account is obtained. Laya is **not** a dependency of the trading
engine — only of this adapter. See [[26 - Decisions]] (D018) for why Jev is a reference
rather than a dependency.

## Hardware notes (RTX 3060 Ti 8 GB)

| Component | VRAM |
|---|---|
| Laya English, bf16 + activations | **~1.2 GB** |
| Main AI (8B Q4) + KV cache | **~6–6.5 GB** |

Together ≈ 7.8 GB of 8 GB. Too tight. **Recommendation: run Laya on CPU.**
It is small, and our event volume does not need GPU throughput. Keep VRAM for the Main AI.
Benchmark actual latency in Phase 5 rather than assuming — see [[21 - Testing]].

## Question battery — design constraints

A battery is a small fixed set of questions. Per the spec, **not dozens of redundant
questions**; each needs a stated purpose.

Candidate battery (validate each before adoption):

| # | Question | Type | Purpose |
|---|---|---|---|
| 1 | Market regime | choice | context filter |
| 2 | Direction | choice | directional lean |
| 3 | Buying pressure | noul | flow condition |
| 4 | Selling pressure | noul | flow condition |
| 5 | Flow quality | choice | strength/confluence |
| 6 | Setup quality | score | entry desirability |
| 7 | Liquidity condition | choice | execution feasibility |
| 8 | Risk state | choice | whether to stand down |
| 9 | Opportunity state | choice | gate for escalating to Main AI |

**Rules:**
- ≤ 3–5 options per choice question.
- ≤ 64 questions total per request (server limit).
- Instrument + raw state share one budget — keep compact.
- Every question must be **backtested for lift** before gating on it. See [[13 - Backtesting]].
- Questions 3/4 are complementary; consider whether both earn their place.

## Testing requirements (Phase 5)

- Adapter returns typed values for each question type.
- `score` is converted to a label by argmax.
- Laya unreachable → `LayaUnavailable`, **no trade**.
- Malformed answer → rejected, no trade.
- Latency benchmarked on CPU and GPU; recorded in [[25 - Research Log]].
- Round-trip: `MarketState` → compact representation → Laya, with truncation asserted.

---

[[03 - System Architecture]] · [[04 - Main AI]] · [[07 - Order Flow]] · [[14 - Calibration]] · [[27 - Known Limitations]]

## See also

- [[06 - Market Data]] — where state comes from
- [[26 - Decisions]] — Laya vs Jev, and the vendor-project timeline
- [[29 - Implementation Roadmap]] — Phase 5