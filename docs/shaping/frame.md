---
shaping: true
---

# HADR Monitor — Frame

## Source

From `REQS.md` (initial idea capture, 2026-07-08):

> The idea for the HADR product is in the README.md. The idea is for a HADR
> agent to be monitoring this feed. This should surface up events of
> interest. I am unsure of the rest.

From `README.md`:

> The raw feeds are noisy: hundreds of minor earthquakes a day, duplicate
> alerts, events with no humanitarian impact. Someone — or something — has
> to filter, assess and summarise them before a responder can act. That is
> what this project builds.
>
> A monitoring agent that: watches the live GDACS, USGS and ReliefWeb
> feeds; filters out the noise and assesses what remains: what happened,
> where, how bad, who is affected; publishes a morning situation report to
> `dashboard.html` at 08:30 Singapore time; runs on a schedule, unattended,
> and stays quiet when nothing has changed.

Interview record: `QUESTIONS.md` (grilling session, 2026-07-08). Distilled
context: `CONTEXT.md`. Accepted decisions: `docs/adr/0001–0006`.

## Problem

A duty officer starting shift has no fast, trustworthy answer to "did
anything happen overnight that we need to act on?" The public feeds that
hold the answer are individually noisy (hundreds of minor quakes a day),
mutually redundant (the same quake arrives from two or three feeds under
different identifiers), and unstable after the fact (magnitudes get
revised, events deleted, alert colours escalate). Reading them raw takes
longer than the decision deserves and still risks missing the one event
that matters.

## Outcome

Every morning at 08:30 SGT, `dashboard.html` holds a dated situation
report the officer can scan in ~2 minutes: confirmed headline events with
plain-language impact summaries, a permissive watchlist of borderline
events, explicit corrections to anything previously reported, and
per-feed health — published unattended, every day including quiet ones, so
the officer can always tell "nothing happened" from "the monitor is
broken."
