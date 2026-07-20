# device-profiles

## ADDED Requirements

### Requirement: Multiple named profiles per participant
The system SHALL allow a participant to carry multiple named profiles (e.g.
`fullsize` / `lowtemp`, `summer` / `winter`), each a complete parameterization of the
participant's class, with exactly one profile active at a time.

#### Scenario: Heat pump summer profile
- **GIVEN** the heat pump's `summer` profile is active
- **WHEN** the engine plans the day
- **THEN** it schedules one uninterrupted DHW cycle instead of the winter all-day
  behavior

> Source: mstormi ([5016228379](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5016228379)),
> seconded from a second installation by masipila
> ([5017073428](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5017073428)),
> program selection in jlaur's founding comment
> ([1481931141](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931141)).

### Requirement: Profile contents
Each named profile SHALL be able to carry a load curve as a TimeSeries (of any range and
interval spacing) together with a scale factor, plus the class parameters it overrides
(demand, deadline, protections), so the same representation covers consumer programs and
seasonal producer curves.

#### Scenario: Washing program curve
- **WHEN** the `fullsize` profile carries its measured curve (peak 1:30 in, one hour
  long) and a scale factor
- **THEN** window cost is computed over that curve, not over a flat average

#### Scenario: Range and intervals suit the device
- **GIVEN** a white-goods profile spanning a day at 1-minute intervals and a heat-pump
  profile spanning a year with mixed intervals
- **WHEN** each profile is scheduled
- **THEN** both are carried as a single variable-interval TimeSeries sized to the
  device's cycle

> Source: "The TimeSeries and the scale factor should be part of the profile"
> ([5016228379](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5016228379));
> range/interval flexibility, mixed intervals in one series (white goods a day, car a
> week or two, heat pump a year) from mstormi
> ([5019271702](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5019271702));
> per-level PowerProfiles (lsiepel,
> [1481931303](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931303));
> curve shape precedent: jlaur's mapped dishwasher program
> ([1481931141](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931141)).

### Requirement: Profiles are updatable at any time
A profile's TimeSeries SHALL be read-write and updatable at any time, so a curve can be
refined or replaced live — by the learning layer or by richer telemetry — without
redefining the profile.

#### Scenario: Learned curve overwrites the baseline
- **GIVEN** a profile carrying an initial curve
- **WHEN** the learning layer produces a better curve from recorded runs
- **THEN** the profile's series is overwritten in place and the next plan uses it

> Source: mstormi ([5020830338](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5020830338));
> ties to the learning layer (wave 4).

### Requirement: Active-profile selection
The system SHALL support selecting the active profile manually, by schedule/season, and
programmatically (rules or detection), with the switch taking effect at the next
planning cycle.

#### Scenario: Season flips the profile
- **WHEN** the configured heating season ends
- **THEN** the heat pump's active profile changes from `winter` to `summer` without
  user action

#### Scenario: Detected program
- **WHEN** an appliance's own state reports which program the user chose
- **THEN** a rule can set the matching named profile before the engine plans it

> Source: seasonal examples in [5016228379](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5016228379)
> and [5017073428](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5017073428);
> "select the last run specific program"
> ([1481931141](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931141)).
