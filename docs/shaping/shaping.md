---
shaping: true
---

# HADR Monitor v1 — Shaping

Frame: `frame.md`. Interview record: `../../QUESTIONS.md`. Distilled
context: `../../CONTEXT.md`. Decisions with rationale: `../adr/`.

This shaping doc composes the grilling session's decisions into a single
shape and checks it against the requirements. The component alternatives
below were considered and rejected during grilling; they are kept as the
audit trail.

---

## Requirements (R)

| ID | Requirement | Status |
|----|-------------|--------|
| R0 | A duty officer deciding activation/escalation reads a severity-first situation report at 08:30 SGT, scannable in ~2 minutes, covering GDACS + USGS + ReliefWeb, all hazards, global | Core goal |
| R1 | Obvious noise is cut by a deterministic, permissive rule floor — every exclusion below the floor is explainable by a named rule, and missing a disaster is treated as worse than noise | Must-have |
| R2 | Surviving candidates are assessed and summarised in plain language (what, where, how bad, who is affected) and tiered into headline / watchlist / drop | Must-have |
| R3 | One physical event appears once — records from multiple feeds merge, with conflicting facts resolved by source domain and per-source attribution kept | Must-have |
| R4 | Changes to previously reported events are called out explicitly (magnitude revisions, deletions, colour escalations), and an escalated event re-qualifies even if previously filtered out | Must-have |
| R5 | A quiet day still publishes a dated "no significant events" report with feed health; published reports are immutable and archived | Must-have |
| R6 | Runs unattended on a schedule with no always-on personal machine; a down feed produces a partial report with a prominent banner, never a missing report | Must-have |
| R7 | State and audit trail are human-inspectable and diffable (plain files in the repo, git history) | Must-have |
| R8 | The LLM is confined to one seam behind a hosted endpoint (URL/api_key/model as config, not code); an LLM failure degrades to rules-only tiering and never blocks the 08:30 report | Must-have |

Constraint noted from grilling (lives in mechanisms, not R): ReliefWeb's
JSON API needs a not-yet-approved appname — v1 uses the no-auth RSS feed
(ADR-0004, QUESTIONS.md I3).

---

## CURRENT: Starter repo

No pipeline exists. The repo holds feed documentation (`feeds/*.md` with
verified endpoints and open questions), course scaffolding, and the docs
produced by grilling. Nothing fetches, filters, or publishes.

## A: Deterministic daily pipeline with one LLM seam

The composite selected through grilling (ADRs 0001–0006): a Python
pipeline on a GitHub Actions cron — fetch once, filter by rules, merge
into canonical events, diff against state, assess via LLM, render, commit.

| Part | Mechanism | Flag |
|------|-----------|:----:|
| **A1** | **Fetchers** — one module per source, each with bounded retry + backoff, writing latest raw payload to `state/raw/` | |
| A1.1 | USGS: GET `all_day.geojson` (rolling 24 h window) | |
| A1.2 | GDACS: GET `geteventlist/EVENTS4APP` (current event list, all hazards) | |
| A1.3 | ReliefWeb: GET `disasters/rss.xml`, parse items + GLIDE tag from description HTML | |
| **A2** | **Rule floor** — named tunable constants producing Candidates: GDACS Orange/Red → headline candidate; USGS mag ≥ 5.5 OR alert ≠ null OR sig ≥ 600 → candidate; new ReliefWeb disaster → candidate; GDACS Green quakes below USGS floor → dropped | |
| **A3** | **Canonical event store & merge** — match records by GLIDE (when present) else same hazard within ~100 km and ~1 h (haversine + time delta); merged CanonicalEvent keeps per-source attribution and all source IDs; conflicts resolved USGS→science, GDACS→alert/impact, ReliefWeb→narrative; persisted in `state/events.json` | |
| **A4** | **Change detection** — diff this run's merged events against `state/events.json`: new / revised (mag, location) / deleted / escalated (episodealertlevel rose); escalation past the floor re-qualifies a previously dropped event | |
| **A5** | **LLM assessment seam** — one function: capped worst-first candidate batch (~40) → OpenCode Zen `/v1/messages` (Anthropic-style, `qwen3.7-max`, auto tool choice on a `submit_assessments` tool with the assessment schema as `input_schema`, via requests/httpx) → validated {tier, summary} per event; one retry with validation errors appended, then rules-only tiering + note in Feed health | |
| **A6** | **Renderer** — build `dashboard.html` (Headline → Watchlist → Updates & corrections → Feed health → generated-at) and archive copy `reports/YYYY-MM-DD.html`; dashboard links to archive; all-quiet variant when no candidates | |
| **A7** | **Scheduled workflow** — GitHub Actions cron (UTC, margin before 08:30 SGT) runs A1→A6 and commits report + archive + state; `OPENCODE_API_KEY` as Actions secret, endpoint/model as config | |

**A5 flag resolved (2026-07-08):** Spike S1 (`spike-llm-endpoint.md`)
verified the OpenCode Zen endpoint with `qwen3.7-max`: 4/4 trials returned
schema-valid tool-use output at the 40-candidate batch size (59–98 s,
well inside the job budget). Caveats folded into A5's mechanism: forced
tool choice unsupported (use auto + prompt mandate), stdlib urllib blocked
(use requests/httpx), thinking model needs generous `max_tokens`.

### Alternatives considered and rejected (audit trail)

| Component | Rejected alternative | Why rejected | Record |
|-----------|---------------------|--------------|--------|
| A2+A5 | LLM judges every raw record | Costly, slow, exclusions unexplainable | ADR-0001 |
| A2+A5 | Rules only, no LLM | Blind to context (M5.2 under a dense city); no plain-language summaries | ADR-0001 |
| A3 | GLIDE-only matching | USGS never carries GLIDE; GDACS's often empty early — misses most merges | ADR-0002 |
| A3 | No dedup, report per feed | Same quake appears 2–3×, fails the 2-minute scan | ADR-0002 |
| A6 | Don't touch dashboard on quiet days | Stale page indistinguishable from dead agent | ADR-0003 |
| A6 | Single rolling page, no archive | Loses day-over-day record; corrections lose their referent | ADR-0003 |
| A7 | Local machine cron | Reports stop when the laptop is off — weak "unattended" | ADR-0004 |
| A7 | Continuous poller + 08:30 renderer | Needs always-on process; single daily fetch suffices (USGS all_day covers 24 h) | ADR-0004 |
| A7 | Hold report until all feeds respond | Officer may get no report at all — worse than a flagged partial | ADR-0004 |
| A3/A4 state | SQLite in repo | Binary diffs poorly; overkill for tens of events | ADR-0005 |
| A3/A4 state | Stateless re-derive | Corrections/escalation detection fragile | ADR-0005 |
| A5/A7 | Claude Code headless as the whole agent | Non-deterministic, hard to test; weak trust story for Day 3 | ADR-0006 |

---

## Fit Check: R × A

| Req | Requirement | Status | A |
|-----|-------------|--------|---|
| R0 | A duty officer deciding activation/escalation reads a severity-first situation report at 08:30 SGT, scannable in ~2 minutes, covering GDACS + USGS + ReliefWeb, all hazards, global | Core goal | ✅ |
| R1 | Obvious noise is cut by a deterministic, permissive rule floor — every exclusion below the floor is explainable by a named rule, and missing a disaster is treated as worse than noise | Must-have | ✅ |
| R2 | Surviving candidates are assessed and summarised in plain language (what, where, how bad, who is affected) and tiered into headline / watchlist / drop | Must-have | ✅ |
| R3 | One physical event appears once — records from multiple feeds merge, with conflicting facts resolved by source domain and per-source attribution kept | Must-have | ✅ |
| R4 | Changes to previously reported events are called out explicitly (magnitude revisions, deletions, colour escalations), and an escalated event re-qualifies even if previously filtered out | Must-have | ✅ |
| R5 | A quiet day still publishes a dated "no significant events" report with feed health; published reports are immutable and archived | Must-have | ✅ |
| R6 | Runs unattended on a schedule with no always-on personal machine; a down feed produces a partial report with a prominent banner, never a missing report | Must-have | ✅ |
| R7 | State and audit trail are human-inspectable and diffable (plain files in the repo, git history) | Must-have | ✅ |
| R8 | The LLM is confined to one seam behind a hosted endpoint (URL/api_key/model as config, not code); an LLM failure degrades to rules-only tiering and never blocks the 08:30 report | Must-have | ✅ |

**Notes:**
- R2 flipped ❌→✅ on 2026-07-08 when Spike S1 verified the endpoint
  (4/4 schema-valid trials at the 40-candidate batch size). Shape A now
  passes the full fit check with no flags.
- R8's wording generalised: the seam is Anthropic-style Messages, not
  OpenAI-compatible — Qwen models on OpenCode Zen only speak the former
  (ADR-0006 as amended). Confinement + degradation demands unchanged.

## Unsolved

1. Threshold constants (A2) and match windows (A3) are heuristics to
   revisit after a week of real reports — tracked in CONTEXT.md, not
   blocking.

## Next

Shape A is selected, unflagged, and passes the fit check. Breadboard it
(`/breadboarding`) into UI / non-UI affordances and wiring, then slice.
Slice 1 is already agreed (QUESTIONS.md J2): USGS → rules → LLM →
dashboard, run by hand.
