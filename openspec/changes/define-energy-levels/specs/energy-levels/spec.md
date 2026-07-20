# energy-levels

## ADDED Requirements

### Requirement: Four-level scale
The system SHALL express energy availability as an ordered four-level scale —
blocked / normal / encouraged ("low price") / overcapacity — matching the Smart-Grid
(SG-ready) industry model so that heat pumps, EVCC-style charge controllers and simple
loads can consume it natively.

#### Scenario: SG-ready mapping
- **WHEN** the site level changes to overcapacity
- **THEN** an SG-ready heat pump participant can be driven to its corresponding mode
  without translation logic in user rules

#### Scenario: Binary collapse
- **WHEN** a Simple ON/OFF consumer subscribes to levels
- **THEN** the four levels collapse to allow/deny for it

> Source: mstormi's Energieniveau model ([1481931249](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931249),
> [docs](https://storm.house/docs/#Energieniveaumanagement)), masipila's SG-mode framing
> ([1481931320](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931320)),
> EVCC mode equivalence ([1481931278](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931278)).

### Requirement: Level derivation
The system SHALL derive the level from the price schedule (base level) and escalate it
on live PV excess, with the number of hours assigned to each non-normal level
user-configurable — including via rules, so e.g. cold weather can reduce blocked hours.

#### Scenario: Cheapest-hours classification
- **GIVEN** a user configuration of 4 overcapacity, 4 low-price and 4 blocked hours
- **WHEN** tomorrow's 24 prices arrive
- **THEN** the planned levels mark the 4 cheapest hours overcapacity, the next 4
  low-price, the 4 most expensive blocked, and the rest normal

#### Scenario: PV escalation
- **GIVEN** an hour whose planned level is normal
- **WHEN** live PV surplus appears
- **THEN** the current level escalates for the duration of the surplus

> Source: mstormi's base-plus-escalation with user-configurable hours ("that's key to
> applicability", [1481931249](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931249)),
> masipila's algorithm sketch incl. rule-updatable hour counts
> ([1481931363](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931363)).

### Requirement: Planned schedule vs. current level
The system SHALL keep the planned level schedule (a future-timestamped TimeSeries) and
the current level (an Item) as separate artifacts, where live conditions may override
the plan for the current slot.

#### Scenario: Plan says normal, sun says overcapacity
- **GIVEN** a planned series holding "normal" for the current hour
- **WHEN** PV excess is live
- **THEN** the current-level Item reads overcapacity while the stored plan stays intact

> Source: masipila's planned-mode / current-mode decoupling
> ([1481931320](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931320),
> [1481931363](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931363)).

### Requirement: Window strategies
The level classifier SHALL support both non-consecutive selection (N cheapest slots) and
consecutive windows (for loads that must not be interrupted), independent of the market
time resolution (60- or 15-minute slots).

#### Scenario: Consecutive window at 15-minute resolution
- **WHEN** prices arrive in 15-minute slots and a consumer needs 2 uninterrupted hours
- **THEN** the classifier finds the cheapest consecutive 8-slot window

> Source: masipila's consecutive vs. non-consecutive discussion and 15-minute market
> outlook ([1481931313](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931313),
> [1481931320](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931320)).

### Requirement: Seasonal window defaults
Level derivation SHALL support seasonal parameter sets (e.g. more encouraged hours in
winter, fewer in summer) selectable automatically by date, as production experience
shows one static configuration does not fit the year.

#### Scenario: Winter widens cheap windows
- **WHEN** seasonal mode is on and the date enters winter
- **THEN** the configured winter hour-counts replace the summer ones without user action

> Source: seasonal profiles raised by mstormi ([5016228379](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5016228379));
> percentile+seasonal windows field-tested in the reference
> ([EMS_ENGINE](https://github.com/stamateviorel/openhab-binding-emsmanager/blob/main/docs/EMS_ENGINE.md)).
