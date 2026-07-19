# grid-constraints

## ADDED Requirements

### Requirement: Peak-based fee models
The system SHALL model grid fees that depend on demand peaks, covering at least the two
documented archetypes: billed maximum of fixed-size power buckets per period (e.g.
Belgian 15-minute monthly peak) and tiered fees on the average of the N highest slots
per period (e.g. Norwegian top-3-hours), so engines can weigh peak cost against price
cost.

#### Scenario: Established peak as budget
- **WHEN** this month's billed 15-minute peak is already 4.2 kW
- **THEN** the engine may schedule freely up to that level for the rest of the month,
  since staying under it adds no capacity cost

#### Scenario: Avoiding a tier jump
- **WHEN** one more high-consumption hour would lift the month's top-3-hour average into
  the next fee tier
- **THEN** the engine can trade that hour against higher-priced but flatter scheduling

> Source: mherwege ([1481931336](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931336)),
> seime ([1481931351](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931351));
> monthly-peak budget mechanics field-tested in the reference (capacity controller).

### Requirement: Load balancing under a power budget
The system SHALL schedule consumers under a total power budget using their declared
maximum power and priority, so mutually exclusive loads are serialized onto the best
slots instead of overlapping.

#### Scenario: Boiler and heater never together
- **WHEN** a 3 kW boiler and 9 kW heating both need hours and the budget is 10 kW
- **THEN** the higher-priority load gets the best window and the other gets the next
  best non-overlapping one

> Source: masipila's multi-objective sketch with maxPower + schedulingPriority
> ([1481931363](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931363));
> seime's limit-load-within-window need
> ([1481931351](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931351)).

### Requirement: Region-configurable definitions
Constraint/fee definitions SHALL be configurable per region and date-dependent, and
SHALL be updatable outside the core release cycle (e.g. as data files or add-ons),
because these rules change on regulatory timetables, not openHAB's.

#### Scenario: Tariff rule changes mid-year
- **WHEN** a grid operator changes its peak-fee formula effective a given date
- **THEN** the user updates the constraint definition without waiting for an openHAB
  release

> Source: seime's region/date-based downloadable configuration
> ([1481931351](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931351));
> mherwege's "engine needs flexibility to insert specific logic"
> ([1481931336](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931336)).
