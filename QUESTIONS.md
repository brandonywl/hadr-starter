# QUESTIONS — HADR Monitor grilling session

Status legend: ⬜ open · 🟨 asked · ✅ answered (answer noted inline) · ❌ dropped

Questions are grouped by domain area. New questions get appended as they
surface during the interview.

## A. Audience & purpose

- ✅ A1. Who is the primary reader of the 08:30 report — an individual duty
  officer, a team, or "future you" grading the course? What decision do they
  make after reading it?
  **Answer:** A duty officer at shift start deciding whether to activate/
  escalate. Report must be scannable in ~2 minutes, severity-first.
- ✅ A2. What does the reader do when the report shows something big — is the
  dashboard the end of the pipeline, or does it need to feed a downstream
  action (tasking, notification, escalation)?
  **Default (unasked):** Dashboard is the end of the pipeline in v1. No
  notifications/escalation hooks; each event links to its source reports so
  the officer can go deeper.
- ✅ A3. Is this product judged on realism (would a real HADR cell use it) or
  on demonstrating the course skills (plan/autonomy/trust)? Where do we cut
  corners when those conflict?
  **Default (unasked):** Resolved implicitly by A1 — build for the duty
  officer persona; course artefacts fall out of doing that properly.

## B. Scope

- ✅ B1. Hazard scope: all GDACS hazard types (EQ, TC, FL, VO, DR, WF) from
  day one, or earthquakes first (the only hazard all three feeds cover)?
  **Answer:** All hazards GDACS carries, from day one.
- ✅ B2. Geographic scope: global, or a region of interest (the 08:30 SGT
  deadline hints at Asia-Pacific)?
  **Answer:** Global. The filter does the narrowing, not the scope.
- ✅ B3. Time scope of a report: strictly "what changed in the last 24h", or
  also ongoing situations (a flood from last week that's still evolving)?
  **Default (unasked):** Last-24h delta is the spine; ongoing situations
  surface through the Updates & corrections section (escalations, revisions)
  rather than being re-reported in full each day.

## C. "Events of interest" — the filter

- ✅ C1. What is the floor for inclusion? E.g. GDACS Orange/Red always in,
  Green out — or is it impact-based (population affected) rather than
  level-based?
  **Answer (confirmed thresholds, tunable constants):** GDACS Orange/Red →
  headline candidates; USGS mag ≥ 5.5 OR alert ≠ null OR sig ≥ 600 →
  candidates; any new ReliefWeb disaster → candidate; GDACS Green quakes
  below USGS thresholds → dropped. LLM sorts candidates into
  headline / watchlist / drop. Revisit after a week of real reports.
- ✅ C2. Is filtering rule-based (thresholds on alertlevel/mag/sig), LLM-based
  (Claude judges humanitarian relevance), or rules-then-LLM?
  **Answer:** Rules then LLM — deterministic floor cuts obvious noise, LLM
  assesses the survivors for humanitarian impact and writes the summary.
- ✅ C3. False positive vs false negative: which failure is worse for the
  reader — a noisy report or a missed disaster? (Sets where thresholds sit.)
  **Answer:** Missing a disaster is worse. Thresholds err permissive;
  borderline events get a one-line mention in a lower section, not dropped.

## D. Event identity & dedup (domain model core)

- ✅ D1. When GDACS and USGS both report the same quake, what makes two
  records the same event — GLIDE numbers, geo+time proximity, magnitude match?
  **Answer:** Geo+time proximity (~100km, ~1h) plus GLIDE when present.
  Records merge into one canonical Event with per-source attribution.
- ✅ D2. Which feed wins when they disagree (magnitude, location, alert level)?
  Is there a source-of-truth ranking?
  **Answer:** USGS wins for quake science (magnitude/location), GDACS for
  alert level/impact estimate, ReliefWeb for humanitarian narrative.
- ✅ D3. GDACS has events *and* episodes (a cyclone gets many episodes). Is our
  unit of reporting the event, the episode, or a day's delta of an event?
  **Default (unasked):** Unit of reporting is the canonical Event. Episode
  data is used only to detect the *current* severity (episodealertlevel) and
  to notice escalations between runs.

## E. Revisions & corrections

- ✅ E1. USGS revises magnitude/location and sometimes deletes events. If
  yesterday's report said M7.1 and it's now M6.4, does today's report issue a
  correction, silently reflect the new value, or both?
  **Answer:** Today's report carries an explicit "Updates & corrections"
  section (revisions, deletions, escalations). Past reports are never edited.
- ✅ E2. Can an event's GDACS colour escalate after we reported it (Green →
  Orange)? Does escalation re-qualify a previously-filtered event?
  **Answer:** Yes — an escalated event re-enters the report even if it was
  filtered out on a previous day.

## F. Report & dashboard

- ✅ F1. What does "stays quiet when nothing has changed" mean concretely —
  don't rewrite dashboard.html at all, or publish a dated "no significant
  events" report?
  **Answer:** Always publish at 08:30 — a timestamped "no significant events"
  report with feed-health status, so a stale page is distinguishable from a
  dead agent.
- ✅ F2. Is dashboard.html one rolling page (latest report only) or does it
  keep history / an archive of past reports?
  **Answer:** dashboard.html shows the latest report, with links to past
  daily reports kept as static files (e.g. reports/2026-07-08.html).
- ✅ F3. What sections must a situation report have? (e.g. new events,
  escalations, corrections, ongoing watch list, feed-health note)
  **Answer:** Headline events (strict tier) · watchlist (permissive tail) ·
  updates & corrections · feed health · generated-at timestamp.
- ⬜ F4. Where is dashboard.html served from — just a file in the repo,
  GitHub Pages, or opened locally?

## G. Runtime & operations

- ✅ G1. Where does the scheduled run execute — this machine (cron / Claude
  Code scheduled agent), GitHub Actions, or something else?
  **Answer:** GitHub Actions scheduled workflow; report committed to the repo.
  No always-on machine.
- ✅ G2. Polling model: poll feeds continuously through the day and accumulate
  state, or one big fetch shortly before 08:30?
  **Answer:** One fetch per feed shortly before 08:30 (USGS all_day covers
  24h; GDACS current event list; ReliefWeb RSS). No continuous poller in v1.
- ✅ G3. Feed down at report time: what does the report say, and do we retry?
  **Answer:** Retry each feed a few times with backoff; if still down,
  publish a partial report from the remaining feeds with a prominent
  "unreachable since …" banner in feed health.
- ✅ G4. Polite polling frequency per feed (GDACS publishes no limits; USGS
  regenerates every minute; ReliefWeb monitors usage)?
  **Answer:** Moot for v1 — one fetch per feed per day (plus bounded
  retries), which is well inside any polite limit.
- ✅ G5. (new) The LLM is a self-hosted vLLM endpoint. Is it reachable from
  GitHub-hosted Actions runners, or does the job need a self-hosted runner /
  local cron? Which model is served, and what context window?
  **Answer (updated 2026-07-08):** OpenCode Zen selected, model
  `qwen3.7-max`, via the Anthropic-style `/zen/go/v1/messages` endpoint;
  key in `.env` as `OPENCODE_API_KEY` (git-ignored) and as an Actions
  secret in CI. Verified fit by Spike S1
  (`docs/shaping/spike-llm-endpoint.md`).

## H. Data & state

- ✅ H1. Where does state live (seen events, published reports, dedup index) —
  flat JSON files, SQLite, or none (stateless re-derive each run)?
  **Answer:** JSON files in the repo (e.g. state/events.json), committed by
  each run. Git history is the audit trail.
- ✅ H2. Do we keep raw feed snapshots for debugging/audit, and for how long?
  **Default (unasked):** Keep the latest raw fetch per feed under state/raw/,
  overwritten each run. Older snapshots live in git history — no separate
  retention machinery.

## I. Tech & constraints

- ✅ I1. Language/stack preference? (CLAUDE.md is currently empty.)
  **Answer:** Python. Deterministic pipeline (fetch → rules → dedup →
  render); LLM confined to one assessment/summarisation seam.
- ✅ I2. Is the "agent" LLM-driven at runtime (Claude assesses/summarises each
  run, costs tokens nightly) or is the LLM only used to build it, with the
  runtime being plain code?
  **Answer:** LLM at runtime for the assessment step — via the user's
  self-hosted vLLM OpenAI-compatible endpoint (OpenAI SDK, custom base_url),
  not the Anthropic API.
- ✅ I3. ReliefWeb appname approval is pending/slow — build against RSS only
  for now?
  **Answer:** Yes — build on the no-auth RSS feed now; the API is a later
  upgrade once the appname is approved.
- ✅ I4. Any budget ceiling per run (API tokens, runtime minutes)?
  **Default (unasked):** Self-hosted LLM ⇒ no token cost pressure. Bound the
  run instead: cap candidates sent to the LLM per run (e.g. 40, worst-first)
  and keep the whole job under ~10 minutes including retries.

## J. Course fit

- ✅ J1. Which artefact does this grilling feed first — prd.html? Should
  CONTEXT.md be written so the PRD can be generated from it?
  **Default (unasked):** Yes — CONTEXT.md is written to be the single input
  from which prd.html and system-view.html can be generated.
- ✅ J2. What is the first vertical slice you'd want on Day 2 — the smallest
  end-to-end path (one feed → filter → dashboard)?
  **Answer:** USGS → rules → LLM → dashboard.html, run once by hand.
  Exercises every layer including the LLM seam; other feeds, dedup and the
  schedule come in later slices.
