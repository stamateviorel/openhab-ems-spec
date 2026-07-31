# optimization-objectives

## ADDED Requirements

### Requirement: Selectable objective

The system SHALL let the user select the optimization objective — at minimum lowest
cost, maximum self-consumption, and lowest carbon — instead of assuming lowest cost.

#### Scenario: Self-consumption over cheap grid

- **GIVEN** the self-consumption objective and a live PV surplus during a cheap grid hour
- **WHEN** a deferrable load is planned
- **THEN** it is placed into the surplus rather than into the cheap grid hour

#### Scenario: Carbon over price

- **GIVEN** the carbon objective and a green-share series showing a very renewable
  afternoon that is slightly pricier than the night
- **WHEN** the same load is planned
- **THEN** it is placed into the greener afternoon

> Source: lsiepel — "look for cheapest or for max self provided or whatever user defined
> preferences" ([1481931215](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931215));
> jlaur — "prioritize Planet Earth over price"
> ([1482016387](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1482016387)).

### Requirement: Carbon data as a first-class series

The system SHALL consume CO₂-intensity or green-share data as future-timestamped series
through the same data plane as prices and forecasts, from pluggable sources.

#### Scenario: Green-share series drives ranking

- **GIVEN** a contributed source publishing tomorrow's renewable share per hour
- **WHEN** the carbon objective ranks hours
- **THEN** the ranking uses that series exactly as the cost objective uses the price
  series

> Source: jlaur — CO₂/green datasets exist and "do not necessarily map directly to the
> prices" ([1482016387](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1482016387));
> storage model shared with `price-data`/`forecast-data`; emissions-provider prior art in
> the reference (fixed factors / live grid-intensity source).

### Requirement: Objective extensibility

The objective SHALL be an extension point, so contributed algorithms and scripts can add
objectives beyond the built-in three without core changes.

#### Scenario: Contributed objective

- **GIVEN** a script contributing a custom objective (e.g. battery-wear-aware cost)
- **WHEN** the user selects it
- **THEN** planning uses it under the same engine guardrails as the built-ins

> Source: lsiepel — "or whatever user defined preferences"
> ([1481931215](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931215));
> mechanism via `extension-surface` and scripts-first-class in `engine-contract`.
