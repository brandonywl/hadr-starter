# ADR-0003: Always-publish daily SitRep with corrections and archive

Date: 2026-07-08 · Status: Accepted

## Context

The README says the agent "stays quiet when nothing has changed" — but for
an unattended system, a dashboard that didn't change is indistinguishable
from an agent that died. Separately, events change after publication: USGS
revises magnitudes and occasionally deletes events; GDACS colours escalate
(Green → Orange). A duty officer acting on yesterday's number needs to be
told it moved.

## Decision

- **Publish every day at 08:30 SGT**, including quiet ones. An uneventful
  morning produces a timestamped "no significant events in the last 24h"
  report with feed-health status. "Stays quiet" means the *content* is
  quiet, not the channel.
- **SitReps are immutable once published.** `dashboard.html` always shows
  the latest; past days are archived as static files
  (`reports/YYYY-MM-DD.html`) linked from the dashboard.
- Fixed section order: **Headline events** (strict tier) → **Watchlist**
  (permissive tail, one line each) → **Updates & corrections** →
  **Feed health** → generated-at timestamp.
- **Updates & corrections** explicitly calls out magnitude/location
  revisions, event deletions (false alarms), and colour escalations against
  what a previous SitRep said.
- **Escalation re-qualifies:** an event filtered out on a previous day
  re-enters today's report if its severity rises past the floor.

## Consequences

- Trust: the officer can always distinguish "nothing happened" from "the
  agent is broken", and never silently acts on a stale figure.
- Requires remembering what was reported when, at what severity — state
  (ADR-0005).
- The archive grows by one small static file per day; git handles it.
