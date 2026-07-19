# Define grid constraints and peak fees

**Wave 3** — the "local government creativity" layer: costs that depend on *how* you
draw power, not just *when*.

## Why

Price-window optimization alone mis-optimizes wherever grid fees charge for peaks:
Belgium bills the monthly maximum of 15-minute average power (capaciteitstarief),
Norway tiers a fixed fee on the average of the month's 3 highest hours, Germany's §14a
dims large consumers. These change the objective function — sometimes it is cheaper to
spread load into pricier hours than to spike in the cheapest one. Production systems
(reference implementation; masipila's multi-objective sketch) show this is tractable if
the model carries per-consumer power bounds and priorities. Regional rules change often,
so their definitions must be updatable outside the core release cycle.

## What changes

- Define the `grid-constraints` capability: peak-based fee models, load balancing under
  a power budget, and region-configurable constraint definitions.

## Non-goals

- Implementing every regional scheme in core — the requirement is pluggability plus the
  two documented archetypes (bucket-peak, top-N-average).
- Hardware load-guard/breaker protection (safety belongs to the engine contract, not to
  fee modeling — though the same power-budget mechanics serve both).

## Impact

- Extends the engine's objective function; adds a constraint-provider surface.
