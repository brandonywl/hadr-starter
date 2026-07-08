---
shaping: true
---

# A5 Spike: OpenCode Zen endpoint fitness for the assessment seam

**Status: COMPLETE (2026-07-08) — endpoint is fit. A5 flag resolved.**

## Context

Shape A confines the LLM to one seam (A5): a batch of candidate events in,
validated `{tier, summary}` per event out. The user selected **OpenCode
Zen** (`https://opencode.ai/zen/go/...`) with model **qwen3.7-max**. Before
this spike, nothing about the endpoint was verified — this was the only ⚠️
in Shape A and the only ❌ in the fit check (R2).

## Goal

Verify the endpoint can deliver the assessment seam's contract, and
identify what the prompt/validation code must accommodate.

## Questions & Answers

| # | Question | Answer |
|---|----------|--------|
| **A5-Q1** | What model does the endpoint serve, and what protocol? | `qwen3.7-max` is served and live. Qwen models use the **Anthropic-compatible** `/zen/go/v1/messages` endpoint (not OpenAI chat completions). It is a **thinking model** — responses carry a `thinking` block before the answer, inflating output tokens (216 output tokens to say "OK"). `/zen/go/v1/models` lists 20 models. |
| **A5-Q2** | Does it support constrained structured output? | **Forced tool choice (`tool_choice: {"type":"tool"}`) fails** — HTTP 400 "Upstream request failed". **Auto tool choice works reliably**: pass the tool with `input_schema` + a prompt instruction "you MUST call the tool"; the model returns `stop_reason: tool_use` with schema-conforming input. |
| **A5-Q3** | Does a ~40-candidate batch fit in one request within the job budget? | Yes. Three 40-event trials: **59–98 s** per call, ~1.3–5.5 k input tokens, 3.2–5.3 k output tokens. Single-batch is fine inside a ~10-minute job; no chunking needed at the 40 cap. |
| **A5-Q4** | How often does output fail schema validation? | **0 failures in 4/4 trials** (one 3-event, three 40-event): valid tool_use, correct tiers, full event-id coverage, substantive summaries. Keep one retry-with-error-feedback as a safety net; fall back to rules-only tiering after that. |
| **A5-Q5** | Is it reachable for CI, and what failure modes exist? | Public URL; auth via `x-api-key` header on `/messages` (`Authorization: Bearer` also works on `/models`). **Python stdlib urllib is blocked (403) regardless of user-agent** — bot filtering; `requests` and `httpx` both pass, so the httpx-based Anthropic SDK is safe. Usage limits: $12/5 h, $30/week, $60/month; responses report `cost` per call. |

## Sample output (trial 1)

> `{"event_id": "evt-000", "tier": "headline", "summary": "A shallow M6.1
> earthquake struck Mindanao, Philippines, with 2.4 million people in the
> exposed area; GDACS Orange alert issued, though felt reports are still
> coming in as the automatic solution is preliminary."}`

## Resulting seam contract (feeds ADR-0006 as amended)

- Endpoint `https://opencode.ai/zen/go/v1/messages`, header `x-api-key:
  $OPENCODE_API_KEY`, `anthropic-version: 2023-06-01`, model
  `qwen3.7-max` — all as configuration.
- One tool `submit_assessments` with the assessment JSON schema as
  `input_schema`; **no `tool_choice`** (auto); prompt mandates the call.
- HTTP via `requests`/`httpx`/Anthropic SDK — never stdlib urllib.
- `max_tokens` sized for thinking overhead (≥ 16 k for a 40 batch).
- Validate: exactly one `tool_use` block, schema fields, tier enum,
  event-id coverage. One retry with validation errors appended; then
  rules-only fallback + Feed-health note.
- Budget guard: single call per run, 40-candidate cap keeps run cost
  trivially inside the $12/5 h window.
