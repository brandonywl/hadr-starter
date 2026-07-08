# ADR-0002: Canonical events via geo+time matching with ranked sources

Date: 2026-07-08 · Status: Accepted

## Context

The same physical earthquake arrives from GDACS (source: NEIC), USGS, and —
days later — ReliefWeb, under different identifiers. Reporting per feed
would show one quake 2–3 times and fail the 2-minute scan. GLIDE numbers
tie ReliefWeb to GDACS but USGS never carries them and GDACS's is often
empty early on, so GLIDE-only matching misses most merges.

GDACS also distinguishes **events** from **episodes** (a cyclone gets many
episodes, each with its own alert level).

## Decision

- Merge RawRecords into one **CanonicalEvent** when they share a GLIDE
  number, or fall within **~100 km and ~1 hour** of each other for the same
  hazard type. Keep per-source attribution on the merged event.
- On conflicting facts, rank sources by domain:
  - **USGS** wins earthquake science: magnitude, location, depth.
  - **GDACS** wins alert level and impact estimate.
  - **ReliefWeb** wins humanitarian narrative.
- The **CanonicalEvent is the unit of reporting**. Episodes are read only to
  determine current severity (`episodealertlevel`) and to detect escalation
  between runs — they never appear as report line items.
- Store all source identifiers on the event (USGS `ids` list, GDACS
  `eventid`, GLIDE) so later records can attach to it.

## Consequences

- One quake appears once, with the best fact from each source.
- The distance/time windows are heuristics and will occasionally over- or
  under-merge; they are named constants, tunable, and mismerges are visible
  in the per-source attribution.
- Dedup requires remembering past events across runs — state is required
  (ADR-0005).
