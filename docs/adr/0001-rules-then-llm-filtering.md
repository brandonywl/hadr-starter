# ADR-0001: Rules-then-LLM filtering with a permissive floor

Date: 2026-07-08 · Status: Accepted

## Context

The raw feeds are noisy — hundreds of minor earthquakes a day, events with
no humanitarian impact. Something must decide what reaches the duty
officer's 2-minute morning scan. Options considered: pure rule thresholds
(explainable, free, but blind to context — a M5.2 under a dense city stays
out), LLM-judges-everything (maximum judgment, but costly, slower, and hard
to explain exclusions), or a hybrid.

The reader's failure bias is settled: **missing a disaster is worse than
noise** (QUESTIONS.md C3).

## Decision

Filter in two stages:

1. **Deterministic rule floor** cuts the obvious noise. v1 thresholds,
   kept as named tunable constants:
   - GDACS alert **Orange or Red** → headline candidate
   - USGS **mag ≥ 5.5** OR **alert ≠ null** OR **sig ≥ 600** → candidate
   - Any **new ReliefWeb disaster** entry → candidate
   - GDACS Green earthquakes below the USGS floor → dropped
2. **LLM assessment** sorts surviving candidates into **headline /
   watchlist / drop** and writes each summary (what, where, how bad, who is
   affected). Candidates are capped (~40/run, worst-first) to bound the run.

Thresholds err permissive: a borderline event becomes a one-line watchlist
entry, not a silent drop.

## Consequences

- The expensive/judgment step only sees a small candidate pool; runs are
  cheap and every exclusion below the floor is explainable by a rule.
- The rules are the main miss risk — hence set permissive, and revisited
  after a week of real reports.
- The LLM seam is a single, mockable function; the pipeline is testable
  without it (see ADR-0006).
