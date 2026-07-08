# REQS — HADR Monitor (initial idea capture)

## In my own words

> The idea for the HADR product is in the README.md. The idea is for a HADR
> agent to be monitoring this feed. This should surface up events of interest.
> I am unsure of the rest.

## From README.md (the source of the idea)

HADR = Humanitarian Assistance and Disaster Response. Responders' first problem
is knowing what just happened. Situational awareness comes from public disaster
feeds; this repo works with three (documented in `feeds/`):

- **GDACS** — multi-hazard feed (EU/UN), colour-coded alert levels
- **USGS** — real-time earthquake data, regenerated every minute
- **ReliefWeb** — UN OCHA curated humanitarian info, slower but human-verified

The raw feeds are noisy: hundreds of minor earthquakes a day, duplicate alerts,
events with no humanitarian impact. Something has to filter, assess and
summarise them before a responder can act.

The product is a monitoring agent that:

- watches the live GDACS, USGS and ReliefWeb feeds
- filters out the noise and assesses what remains: what happened, where, how
  bad, who is affected
- publishes a morning situation report to `dashboard.html` at 08:30 Singapore
  time
- runs on a schedule, unattended, and stays quiet when nothing has changed

## What is decided vs. not

- **Decided:** agent monitors the feeds and surfaces events of interest.
- **Undecided:** everything else — to be worked out through interview
  (grilling), shaping, and breadboarding.

## Context

Built as part of a 3-day course (Plan / Autonomy / Trust). Expected artefacts
by the end: `prd.html`, `system-view.html`, `implementation-notes.md`,
`dashboard.html`, `goal.md`, and at least one skill.
