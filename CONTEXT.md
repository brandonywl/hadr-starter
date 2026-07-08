# CONTEXT — HADR Monitor

Product context distilled from the grilling session of 2026-07-08.
Source questions and answers: `QUESTIONS.md`. Decisions with rationale:
`docs/adr/`. This file is the single input from which `prd.html` and
`system-view.html` should be generated.

## Product in one paragraph

An unattended monitoring agent that watches three public disaster feeds
(GDACS, USGS, ReliefWeb), filters out the noise, merges duplicate records of
the same physical event, has an LLM assess and summarise what remains, and
publishes a daily situation report to `dashboard.html` at 08:30 Singapore
time — every day, including quiet ones, so a stale page is distinguishable
from a dead agent.

## Reader & purpose

The primary reader is a **duty officer** at shift start deciding whether
anything needs activation or escalation today. Consequences:

- The report must be scannable in **~2 minutes**, severity-first.
- **Missing a disaster is worse than noise.** Every threshold errs
  permissive; borderline events get a one-line watchlist mention rather than
  being dropped.
- The dashboard is the end of the pipeline in v1 — no notification or
  tasking hooks. Each event links to its source reports for deeper reading.

## Scope

- **Hazards:** everything GDACS carries — earthquakes, cyclones, floods,
  volcanoes, drought, wildfires — from day one.
- **Geography:** global. The filter does the narrowing, not the scope.
- **Time:** each report covers the last-24h delta. Ongoing situations
  surface through escalations and corrections, not full re-reporting.

## Domain model

### Glossary

| Term | Meaning |
|---|---|
| **Source** | One of the three feeds: GDACS, USGS, ReliefWeb (RSS in v1). |
| **RawRecord** | One item as fetched from one source, unmodified. |
| **CanonicalEvent** | One physical disaster, merged from 1–3 sources' RawRecords. The unit of reporting. |
| **Episode** | GDACS's sub-unit of an event (a cyclone gets many). Used only to read *current* severity and detect escalation — never reported on directly. |
| **Candidate** | A CanonicalEvent that passed the rule floor and goes to the LLM. |
| **Assessment** | The LLM's verdict on a Candidate: tier (headline / watchlist / drop) + a written summary (what, where, how bad, who is affected). |
| **SitRep** | One day's situation report: dated, immutable once published. |
| **Correction** | A SitRep entry noting that a previously reported fact changed (magnitude revision, deletion, colour escalation). |
| **FeedHealth** | Per-source fetch status for this run, always shown in the SitRep. |

### Identity & merging (ADR-0002)

Two RawRecords describe the same CanonicalEvent when they match on **GLIDE
number** (when present), or fall within **~100 km and ~1 hour** of each
other for the same hazard type. On conflicting facts, sources rank by
domain:

- **USGS** wins on earthquake science (magnitude, location, depth).
- **GDACS** wins on alert level and impact estimate.
- **ReliefWeb** wins on humanitarian narrative.

Per-source attribution is kept on the merged event.

### Filtering (ADR-0001)

Rules first, then LLM. The deterministic floor (tunable constants):

- GDACS **Orange/Red** → headline candidate.
- USGS **mag ≥ 5.5** OR **alert ≠ null** OR **sig ≥ 600** → candidate.
- Any **new ReliefWeb disaster** → candidate.
- GDACS Green earthquakes below the USGS floor → dropped.

The LLM sorts candidates into **headline / watchlist / drop** and writes
each summary. Cap ~40 candidates per run, worst-first.

### Revisions (ADR-0003)

Events change after publication (USGS revises and deletes; GDACS colours
escalate). Published SitReps are never edited. Instead, today's SitRep
carries an **Updates & corrections** section, and an escalated event
**re-qualifies** even if it was filtered out on a previous day.

## The daily SitRep

Published to `dashboard.html` (latest) with past days archived as static
files (`reports/YYYY-MM-DD.html`). Sections, in order:

1. **Headline events** — strict tier, severity-first.
2. **Watchlist** — permissive tail, one line each.
3. **Updates & corrections** — revisions, deletions, escalations.
4. **Feed health** — per-source status; a down feed gets a prominent
   "unreachable since …" banner.
5. **Generated-at timestamp** — always present, even on all-quiet days.

An uneventful morning still publishes a dated "no significant events"
report (ADR-0003).

## Runtime (ADR-0004)

- **GitHub Actions scheduled workflow**, running shortly before 08:30 SGT.
- **Fetch-at-report:** one fetch per feed per run (USGS `all_day` covers
  24h; GDACS current event list; ReliefWeb RSS). No continuous poller.
- **Degradation:** bounded retries with backoff; if a feed stays down,
  publish a partial report and flag it in Feed health.
- **ReliefWeb:** RSS only in v1 (no auth); the API is an upgrade once the
  appname is approved.
- Run budget: whole job under ~10 minutes including retries.

## State (ADR-0005)

JSON files committed to the repo by each run — `state/events.json` (seen
events, what was reported when, at what severity) and `state/raw/` (latest
raw fetch per feed, overwritten each run). Git history is the audit trail.

## Stack & LLM (ADR-0006)

Python pipeline: fetch → rules → dedup → LLM assessment → render HTML.
The LLM is reached through an **OpenAI-compatible endpoint** (self-hosted
vLLM; an OpenCode endpoint is the fallback — either way an internet-
reachable base URL + API key, held as Actions secrets). The LLM is confined
to one seam: assessing candidates and writing summaries. Everything else is
deterministic and testable without it.

## Slice plan

1. **Slice 1 (Day 2 first):** USGS → rules → LLM → `dashboard.html`, run
   once by hand. Exercises every layer including the LLM seam.
2. GDACS + ReliefWeb fetchers; cross-feed dedup and merging.
3. State, corrections, escalation re-qualification.
4. GitHub Actions schedule + degradation behaviour; archive pages.

## Assumptions taken by default (veto anytime)

- Dashboard is the pipeline's end — no notifications in v1.
- Unit of reporting is the CanonicalEvent; episodes only feed severity.
- Raw snapshots: latest-only on disk, history via git.
- No token budget pressure (self-hosted LLM); bound work by candidate cap
  and job time instead.

## Known open items

- Exact LLM model name / context window — treat as configuration.
- ReliefWeb API appname approval — pending; RSS meanwhile.
- Threshold tuning — revisit constants after a week of real reports.
