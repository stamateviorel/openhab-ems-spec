# forecast-data

## ADDED Requirements

### Requirement: Forecasts as future-timestamped series
The system SHALL represent all forecasts (solar production, temperature, cloud cover,
wind, …) as future-timestamped TimeSeries with fresh-overwrites-old semantics, reusing
the same storage capability as prices.

#### Scenario: Refreshed solar forecast
- **WHEN** a newer forecast run publishes values for timestamps already stored
- **THEN** the new values replace the old ones for those timestamps

> Source: masipila's common-nominator analysis
> ([1481931272](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931272),
> [1481931313](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931313));
> Kai's need #1 ([1481931209](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931209)).

### Requirement: Layered prediction series
A prediction series (generation, consumption, or price) SHALL be read-write and updatable
at any time over any of its entries — past, present or future — so a long-range baseline
can be pre-filled and then progressively overwritten by better sources as they arrive,
including a known cap written directly onto the affected entries.

#### Scenario: PV baseline refined by forecast
- **GIVEN** a PV generation series pre-filled for the whole year from daily sunshine
  duration (its daylight window shifting a few minutes per day)
- **WHEN** a solar or weather forecast becomes available for the coming days
- **THEN** those days' entries are overwritten with the forecast while the rest of the
  year keeps the baseline

#### Scenario: Hard generation cap written into today
- **GIVEN** an inverter with a built-in generation limit, or a §14a dimming limit in
  effect, that openHAB cannot otherwise read or control
- **WHEN** the cap applies to part of today
- **THEN** the affected entries of today's generation prediction are overwritten to the
  capped value

#### Scenario: Forecast source fails
- **GIVEN** a series carrying a pre-filled baseline
- **WHEN** the internet or forecast provider is unavailable for any duration
- **THEN** the EMS keeps planning on the baseline instead of losing the series

> Source: mstormi ([5020830338](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5020830338)) —
> read-write TimeSeries, a PV-year baseline refined live, inverter/§14a caps written into
> the prediction, and graceful fallback when live sources fail.

### Requirement: Solar production forecast
The system SHALL provide plant-level PV production forecasts through the common surface,
sourced from pluggable services, at hourly or finer resolution, so engines can plan
PV-first scheduling and charts can overlay forecast ahead of *now*.

#### Scenario: PV-first boiler window
- **WHEN** tomorrow's solar forecast shows a strong midday plateau
- **THEN** the engine can place solar-first demand into that window instead of the
  cheapest grid hours

> Source: mstormi's forecast-service usage + pointer to the solar forecast binding
> ([1481931328](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931328));
> field practice in the reference (forecast series drawn ahead of now,
> [repo](https://github.com/stamateviorel/openhab-binding-emsmanager)).

### Requirement: Derived-demand forecasts
The system SHALL allow demand forecasts to be computed from other forecasts and
published as first-class series — the proven case being heating need derived from
outdoor temperature (approximately linear), passive solar gain (solar forecast as
proxy), and pre-heating ahead of steep temperature drops.

#### Scenario: Cold snap pre-heat
- **WHEN** the temperature forecast shows 0 → −20 °C within 12 hours
- **THEN** the heating-need series rises ahead of the drop so the engine pre-heats in
  the cheaper, warmer hours

> Source: masipila 2026 ([5017073428](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5017073428));
> pre-conditioning planner prior art in the reference (RC model + DP pre-heat).

### Requirement: Source-agnostic consumption
Engines and rules SHALL consume forecast series without knowing which add-on produced
them, selected by user configuration, so forecast sources are swappable.

#### Scenario: Swapping forecast services
- **WHEN** a user replaces one solar-forecast service with another
- **THEN** no engine or rule change is needed — only the source configuration

> Source: provider-owns-complexity principle (lsiepel,
> [1481931283](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931283));
> pluggable-sources intent throughout the 2023 design
> ([1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374)).
