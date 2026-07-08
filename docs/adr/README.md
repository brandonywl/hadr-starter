# Architecture Decision Records

Decisions from the 2026-07-08 grilling session (see `QUESTIONS.md` and
`CONTEXT.md` at the repo root).

- [ADR-0001](0001-rules-then-llm-filtering.md) — Rules-then-LLM filtering with a permissive floor
- [ADR-0002](0002-canonical-event-identity.md) — Canonical events via geo+time matching with ranked sources
- [ADR-0003](0003-daily-sitrep-semantics.md) — Always-publish daily SitRep with corrections and archive
- [ADR-0004](0004-runtime-github-actions-fetch-at-report.md) — GitHub Actions cron, fetch-at-report, partial-publish degradation
- [ADR-0005](0005-state-as-json-in-repo.md) — State as JSON files committed to the repo
- [ADR-0006](0006-python-openai-compatible-llm-seam.md) — Python pipeline; LLM via hosted endpoint at one seam (amended: OpenCode Zen / qwen3.7-max, Anthropic-style API)
