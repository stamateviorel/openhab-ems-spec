# Add the adaptive learning layer

**Wave 4** — settings and profiles that maintain themselves. Deliberately last: it
writes into containers the earlier waves define, and none of them depend on it.

## Why

Two kinds of hand-tuning dominate EMS upkeep. First, physical parameters: how long can
heating stay off before comfort suffers — masipila reports it took two winters to learn
his house. Second, device curves: profiles drift from reality unless re-measured.
@mstormi's vision (2026): a layer that learns the house's heat-loss rate by correlating
weather with indoor temperature, and adapts or replaces consumption TimeSeries profiles
from recorded history. @jlaur wanted the second half in the thread's second comment
("the mapping of the program could be done automatically" from persisted runs). A
working v0 of the first half exists: the reference implementation learns an online RC
thermal model (with error tracking) and plans pre-conditioning with it.

## What changes

- Define the `adaptive-learning` capability: learned thermal/settings models, learned
  load profiles from history, and the guardrails (learning proposes, never silently
  overrides safety).

## Non-goals

- Any specific ML technology — requirements admit anything from regression to whatever;
  the proven baseline is simple online estimation.
- Cloud dependency — learning MUST work on-premise on persisted item data.

## Impact

- Consumes persistence + wave-2 forecasts; writes wave-3 profile containers and
  suggested settings. No earlier wave depends on this one.
