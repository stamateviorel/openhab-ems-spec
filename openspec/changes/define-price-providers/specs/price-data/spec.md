# price-data

## ADDED Requirements

### Requirement: Prices as future-timestamped series
The system SHALL represent energy prices as future-timestamped TimeSeries of
`Number:EnergyPrice` — not as per-slot channels — where a newly published value for a
timestamp overwrites the older value.

#### Scenario: Day-ahead arrival
- **WHEN** tomorrow's 24 (or 96) prices are published by a source
- **THEN** they are stored as one series usable for charts, calculations and level
  derivation alike

> Source: masipila's 96-channels reductio and storage model
> ([1481931313](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931313));
> capability landed in openHAB 4.1 (TimeSeries + `forecast` strategy), EnergyPrice UoM
> from [1484171583](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1484171583).

### Requirement: Price component composition
The system SHALL compute the effective consumer price as a user-configured sum of
components (e.g. spot price, grid tariff, taxes), where each component may come from a
different source, and the user selects which components/Items are included.

#### Scenario: Spot from one binding, tariff from another
- **WHEN** spot prices come from an ENTSO-E source and the grid tariff from a separate
  calculator
- **THEN** window optimization runs on the summed effective price

> Source: jlaur's elements model + "user should be able to configure which items to
> include" ([1482016387](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1482016387),
> [1489220814](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1489220814));
> Kai agreed ([1500300143](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1500300143));
> masipila's two-binding composition ([1492955803](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1492955803)).

### Requirement: Generic grid-price provider
The system SHALL ship one generic, configurable grid-price provider that reads a future
price series from an Item and applies standard adjustments (VAT multiplier, unit
conversion such as EUR/MWh → ct/kWh, fixed fees, simple conditional tariffs), so common
cases need no custom binding.

#### Scenario: ENTSO-E raw to consumer price
- **WHEN** a source publishes EUR/MWh without VAT and the user configures ×VAT and /10
- **THEN** the effective series is in ct/kWh with VAT, without any add-on dependency

#### Scenario: Seasonal/day-night tariff
- **WHEN** the user configures "winter Mon–Sat 07–22 = higher tariff"
- **THEN** the provider composes it onto the spot series correctly

> Source: Kai's `GridEnergyProvider` ([1483930195](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1483930195),
> [1482078918](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1482078918));
> masipila's Caruna season tariff + unit/VAT math
> ([1481931363](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931363),
> [1482324097](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1482324097)).

### Requirement: Shared window calculations
The system SHALL provide shared, reusable price calculations — at minimum: N cheapest /
most expensive slots (consecutive and non-consecutive) and cost of a given load curve in
a given window — implemented once and available to engines, rules and scripts.

#### Scenario: Cheapest consecutive window for a curve
- **WHEN** a rule asks for the cheapest 3-hour consecutive window for a known load curve
  before 07:00
- **THEN** the shared calculation returns start time and expected cost without the rule
  reimplementing the math

> Source: the issue's founding requirement (calculations implemented once,
> [issue body](https://github.com/openhab/openhab-core/issues/3478)); masipila's
> contributed-algorithms concept ([1481931363](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931363)).

### Requirement: Feed-in pricing
The system SHALL model the feed-in (injection) price separately from the consumption
price, including zero and negative effective feed-in prices, so engines can weigh
self-consumption against export honestly.

#### Scenario: Negative feed-in
- **WHEN** delivering to the grid costs more in tariffs than the spot revenue
- **THEN** the engine sees a negative feed-in price and prefers any local consumption

> Source: jlaur's Danish net-tariff case + Kai's "we would need a way to express this"
> ([1482016387](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1482016387),
> [1482078918](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1482078918)).

### Requirement: Time resolution
All price and forecast handling SHALL be resolution-agnostic ("timeResolution", not
"granularity") and accept non-uniform intervals within one series, so a firm near-term
segment and a coarser far-term segment can share a single TimeSeries.

#### Scenario: Market switches to 15 minutes
- **WHEN** a source starts publishing 96 slots/day
- **THEN** composition, calculations and level derivation keep working unchanged

#### Scenario: Mixed intervals in one series
- **GIVEN** a series holding 15-minute day-ahead prices for tomorrow and hourly predicted
  prices for the rest of the week
- **WHEN** calculations run over it
- **THEN** each entry is treated by its own timestamp and interval, with no assumption of
  a fixed slot width

> Source: masipila ([1481931313](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931313),
> naming in [1482324097](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1482324097));
> non-uniform and mixed intervals in one series (day/week/year provider examples) from
> mstormi ([5019271702](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5019271702)).
