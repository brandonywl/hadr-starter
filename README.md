# HADR Monitor

## What is HADR?

**HADR** stands for **Humanitarian Assistance and Disaster Response** — the
coordinated effort to deliver aid and relief when disasters strike: earthquakes,
cyclones, floods, volcanic eruptions, droughts, wildfires. Governments,
militaries, UN agencies and NGOs all run HADR operations, and they all share the
same first problem: **knowing what just happened**.

That situational awareness comes from public disaster feeds. This repository
works with three of them (documented in `feeds/`):

- **GDACS** — the Global Disaster Alert and Coordination System (EU/UN),
  a multi-hazard feed where every event carries a colour-coded alert level
- **USGS** — real-time earthquake data from the United States Geological
  Survey, regenerated every minute
- **ReliefWeb** — UN OCHA's curated humanitarian information service, slower
  but human-verified

The raw feeds are noisy: hundreds of minor earthquakes a day, duplicate alerts,
events with no humanitarian impact. Someone — or something — has to filter,
assess and summarise them before a responder can act. That is what this project
builds.

## What you will build

A monitoring agent that:

- watches the live GDACS, USGS and ReliefWeb feeds
- filters out the noise and assesses what remains: what happened, where, how
  bad, who is affected
- publishes a morning situation report to `dashboard.html` at 08:30 Singapore
  time
- runs on a schedule, unattended, and stays quiet when nothing has changed

How it does any of that is not specified anywhere in this repository. That is
the course.

## The three days

1. **Plan** — interrogate the feeds, write the PRD, cut it into vertical slices
2. **Autonomy** — build the first slice, write a skill, wire up the 08:30 routine, launch the overnight loop
3. **Trust** — review code you didn't write, harden the pipeline, demo

## Artefacts expected by the end

`prd.html` · `system-view.html` · `implementation-notes.md` · `dashboard.html` · `goal.md` · at least one skill

## Day 1 setup

1. Sign in to Claude Code with your Team seat
2. Create your own repository from this template, then clone it
3. Run `/install-github-app` so @claude reviews your pull requests from Day 2
4. Install OpenCode and sign in with your Go key

Fill in `CLAUDE.md` before your first prompt.
