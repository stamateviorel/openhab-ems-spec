# Define the forecast data plane

**Wave 2** — the non-price half of "a mechanism to store future prices *and e.g. also
weather forecast data*" (Kai's need #1).

## Why

Every optimization in the thread consumes forecasts: solar production (PV-first
charging), weather/temperature (heating demand), cloudiness, wind. masipila's 2023
insight still holds: these are all the **same data shape** — future-timestamped series
where fresh values overwrite old ones — and since openHAB 4.1 core can store exactly
that. The 2026 discussion added the unifying corollary: *derived demand* (like heating
need) is itself a forecast series computed from other forecasts. What's missing is the
common provider surface so engines don't care which binding produced the series.

## What changes

- Define the `forecast-data` capability: forecast series semantics, provider surface,
  solar production forecast, and derived-demand forecasts.

## Non-goals

- Learning/adapting forecasts from history (wave 4).
- Prices (previous change; same storage shape, different capability).

## Impact

- Core interface; existing forecast bindings (solar forecast, weather) become sources.
