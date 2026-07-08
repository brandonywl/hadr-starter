# ADR-0006: Python pipeline; LLM via hosted endpoint at one seam

Date: 2026-07-08 · Status: Accepted (amended 2026-07-08 after Spike S1)

> **Amendment (2026-07-08):** originally this ADR assumed an
> OpenAI-compatible endpoint (self-hosted vLLM). The user selected
> **OpenCode Zen with `qwen3.7-max`**, and Qwen models on Zen are served
> only via the **Anthropic-compatible Messages API**. Spike S1
> (`docs/shaping/spike-llm-endpoint.md`) verified the endpoint and set the
> concrete seam contract below. The principle — one seam, config-driven,
> graceful degradation — is unchanged.

## Context

The assessment step (ADR-0001) needs an LLM at runtime. The user's LLM is
OpenCode Zen (`https://opencode.ai/zen/go/v1/messages`, Anthropic-style),
model `qwen3.7-max`, authenticated with `OPENCODE_API_KEY`. An alternative
considered was running Claude Code headless as the whole agent: maximally
agentic, but non-deterministic and hard to test.

Spike S1 findings that constrain the design:

- Forced tool choice (`tool_choice: {"type": "tool"}`) returns HTTP 400 —
  only auto tool choice works.
- `qwen3.7-max` is a thinking model: a `thinking` block precedes the
  answer, so `max_tokens` must be generous (≥16k for a 40-event batch).
- Python stdlib urllib is blocked (403) by bot filtering; `requests` and
  `httpx` pass.
- 4/4 trials returned schema-valid tool-use output at the 40-candidate
  batch size, in 59–98 s per call.
- Usage limits: $12/5h, $30/week, $60/month.

## Decision

- **Python** implements the pipeline: fetch → rules → dedup → LLM
  assessment → render HTML.
- The LLM is confined to **one seam**: a function that takes candidate
  events and returns validated assessments (tier + summary). Everything
  else is deterministic and testable with the seam mocked.
- Seam contract (from Spike S1):
  - POST to the Messages endpoint with headers `x-api-key:
    $OPENCODE_API_KEY`, `anthropic-version: 2023-06-01`; endpoint URL and
    model name are configuration, never hardcoded.
  - One tool `submit_assessments` whose `input_schema` is the assessment
    JSON schema; **no `tool_choice`** — the prompt mandates the call.
  - HTTP via `requests`/`httpx` (or the Anthropic SDK) — never urllib.
  - Validate the returned `tool_use` input: schema fields, tier enum,
    event-id coverage. One retry with validation errors appended; after
    that, degrade to rules-only tiering with a note in Feed health —
    never block the 08:30 report.
  - One call per run, candidates capped at 40 worst-first (ADR-0001) —
    trivially inside the usage limits.

## Consequences

- The single seam keeps the trust story simple for Day 3: everything
  except one function is deterministic and reviewable.
- Auto (not forced) tool choice means a misbehaving model could answer in
  text; the validator + retry + fallback path covers this, and 4/4 spike
  trials complied on the first attempt.
- Thinking-token overhead makes per-call latency ~1–2 minutes at full
  batch — fine for a daily job, unsuitable if we ever need interactive
  latency.
- Switching models later (e.g. one on Zen's OpenAI-style route) means
  touching only the seam's transport, not the pipeline.
