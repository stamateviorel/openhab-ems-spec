# Define the energy-level model

**Wave 1** — the shared "how available is cheap power right now" signal.

## Why

Two production systems (storm.house since years, the emsmanager reference since 2026)
independently run the same abstraction: a small ordered **energy level** computed from
tariff and PV surplus, consumed by every device. It works because the world already
speaks it: SG-ready heat pumps have 4 modes, EVCC has 4 charge modes, and simple
devices collapse it to allow/deny. masipila's spot-price-optimizer produces the same
shape as planned mode schedules. The thread converged on this in 2023
([1481931320](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931320));
it deserves to be a first-class core concept.

## What changes

- Define the `energy-levels` capability: the 4-level scale, how the level is derived,
  per-level user configuration, planned-schedule vs. current-level decoupling, and the
  standard mappings (SG-ready, EVCC, ON/OFF).

## Non-goals

- The price/forecast data the derivation reads (wave 2).
- Per-device behavior at a given level — that is the participant model's job
  (`runAtLevel` gating is already covered there conceptually).

## Impact

- New core concept + one published "site energy level" signal; no existing behavior
  changes.
