# ADR-0004: GitHub Actions cron, fetch-at-report, partial-publish degradation

Date: 2026-07-08 · Status: Accepted

## Context

The agent must run on a schedule, unattended. A local machine (cron or a
scheduled Claude Code agent) stops reporting when the laptop is off — a weak
"unattended" story. A continuous poller catches escalation timelines but
needs an always-on process. The feeds themselves make a single daily fetch
viable: USGS `all_day` is a rolling 24h window, GDACS serves its current
event list, ReliefWeb RSS lists recent disasters.

ReliefWeb's JSON API requires a pre-approved `appname` (form + email
confirmation) that may not arrive during the course week; its RSS feed
needs no approval. GDACS publishes no rate limits or uptime guarantees.

## Decision

- A **GitHub Actions scheduled workflow** runs shortly before 08:30 SGT
  (schedule expressed in UTC; note Actions cron can start late — schedule
  with margin), fetches all feeds once, builds the SitRep, and **commits**
  the report, archive page, and state to the repo.
- **Fetch-at-report:** one fetch per feed per run. No continuous poller in
  v1. This is also the politeness answer — one request per feed per day is
  inside any conceivable limit.
- **Degradation:** each fetch retries a few times with backoff. If a feed
  is still down, publish a **partial report** from the remaining feeds with
  a prominent "unreachable since …" banner in Feed health. Never hold the
  08:30 report hostage to a dead feed.
- **ReliefWeb v1 = RSS.** The JSON API is an upgrade once the appname is
  approved (tracked in QUESTIONS.md I3).

## Consequences

- No infrastructure beyond the repo; state and audit trail ride along in
  git (ADR-0005).
- Escalations that rise and fall between runs are invisible — accepted for
  v1; a poller can be added later without changing the report contract.
- The workflow needs the LLM endpoint's base URL and API key as Actions
  secrets (ADR-0006).
