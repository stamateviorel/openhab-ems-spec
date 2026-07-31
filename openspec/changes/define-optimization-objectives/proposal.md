# Define optimization objectives beyond cheapest

**Wave 2** — the metric the planner optimizes for is a user choice, not a constant.

## Why

The thread never actually agreed that "best" means "cheapest" — the opposite: lsiepel's
engine takes "user set limits and preferences … look for cheapest or for max self
provided or whatever user defined preferences"
([1481931215](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931215));
jlaur pointed at CO₂/green-share datasets that "do not necessarily map directly to the
prices — so if one would prioritize Planet Earth over price, this data could be a driver
for decision-making rather than prices"
([1482016387](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1482016387));
and masipila deliberately renamed "cheapest hours" to "**best** hours" following mstormi
([1481931363](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931363)).
Today every planning requirement in this corpus implicitly assumes cost. This change
makes the objective explicit, selectable and extensible.

## What changes

- Define the `optimization-objectives` capability: a selectable objective (cost /
  self-consumption / carbon at minimum), carbon data as a first-class series, and
  objective extensibility for contributed algorithms.

## Non-goals

- Choosing a default beyond "cost" — that stays a product decision.
- Multi-objective weighting math — parked as an open design question, not required for v1.
- Fee/peak models — `define-grid-constraints` (they apply under any objective).

## Impact

- Parameterizes the planning side of `engine-contract` and the level derivation in
  `energy-levels`; consumes the `forecast-data`/`price-data` planes.
