# ADR-0005: State as JSON files committed to the repo

Date: 2026-07-08 · Status: Accepted

## Context

Dedup (ADR-0002) and corrections/escalation detection (ADR-0003) require
memory across runs: which events have been seen, what was reported when, at
what severity. Options: JSON files in the repo, a SQLite file, or stateless
re-derivation from yesterday's report. The active-event population is tens,
not thousands. The runtime is a fresh Actions runner each day (ADR-0004),
so state must live somewhere the workflow can read and write — the repo is
the natural place.

## Decision

- **`state/events.json`** — the canonical event store: source identifiers,
  merged facts, current severity, and reporting history (which SitRep said
  what).
- **`state/raw/`** — the latest raw fetch per feed, overwritten each run,
  kept for debugging.
- Each scheduled run commits updated state alongside the report. **Git
  history is the audit trail** — no separate retention or snapshot
  machinery.

## Consequences

- Human-inspectable and diffable: every run's state change is a readable
  commit.
- SQLite rejected as overkill at this scale and opaque in diffs; stateless
  rejected because corrections/escalation detection becomes fragile.
- If the event population ever grows past what a single JSON file handles
  comfortably, revisit — the store is behind one module boundary.
