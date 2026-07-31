# Design notes — open questions

## 1. Overwriting past entries (feasibility dependency)

Future entries are native core capability: TimeSeries + the `forecast` persistence
strategy (openHAB 4.1). Overwriting _past or current_ entries — the §14a/inverter cap
written onto today's prediction — means modifying persisted history. Core ships the API
for it: `ModifiablePersistenceService` (`store(Item, ZonedDateTime, State)`,
`remove(FilterCriteria)`), so the layered-prediction requirement is implementable — but
only against persistence services that implement that interface. Open: survey which
services qualify (tasks 2.4), and whether the layered-series surface should require a
modifiable service or offer a fallback (e.g., a cap held in the engine's view without
rewriting stored history).

## 2. Writer precedence on a layered series

Multiple writers target the same series: the baseline generator, the live forecast
refresh, a cap writer, later the learning layer. Two stated behaviours can collide —
"fresh overwrites old" says a forecast refresh replaces entries; the cap scenario says
those same entries hold the capped value. As written, a refresh arriving after the cap
would silently erase it. Options: re-apply caps after any refresh, give writers a
precedence order, or model caps as a separate constraint series composed at read time.
Undecided — flagged for the prototype to surface (tasks 2.3).
