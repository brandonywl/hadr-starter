# ADR-0006: Python pipeline; LLM via OpenAI-compatible endpoint at one seam

Date: 2026-07-08 · Status: Accepted

## Context

The assessment step (ADR-0001) needs an LLM at runtime. The user self-hosts
a vLLM server exposing an OpenAI-compatible API, and will provide an
internet-reachable endpoint + API key either way (vLLM directly, or an
OpenCode endpoint as fallback) — so GitHub-hosted runners can reach it. An
alternative considered was running Claude Code headless as the whole agent:
maximally agentic, but non-deterministic and hard to test.

## Decision

- **Python** implements the pipeline: fetch → rules → dedup → LLM
  assessment → render HTML.
- The LLM is called through the **OpenAI SDK with a custom `base_url`**.
  `base_url`, `api_key`, and `model` are configuration (GitHub Actions
  secrets / environment variables), never hardcoded — swapping vLLM for the
  OpenCode fallback is a config change.
- The LLM is confined to **one seam**: a function that takes candidate
  events and returns structured assessments (tier + summary). Everything
  else is deterministic and testable with the seam mocked.
- The assessment call requests structured output and is validated; a
  malformed or failed LLM response degrades to rules-only tiering with a
  note in Feed health, rather than blocking the 08:30 report.

## Consequences

- The nightly run has no per-token cost pressure (self-hosted); work is
  bounded by the candidate cap and a ~10-minute job budget instead.
- Model quality is whatever the vLLM host serves — the prompt and the
  structured-output contract must be robust to a smaller model than a
  frontier API would give.
- The single seam keeps the trust story simple for Day 3: everything except
  one function is deterministic and reviewable.
