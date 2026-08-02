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

#### Scenario: Below the minimum billable demand there is nothing to shave

- **GIVEN** a tariff carrying a minimum billable demand and a month-to-date peak below it
- **WHEN** the model is asked what the next slot may draw
- **THEN** the answer is the minimum billable demand, because peaks below that floor are
  billed identically and shaving them buys nothing

> Source: mherwege ([1481931336](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931336)),
> seime ([1481931351](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931351));
> monthly-peak budget mechanics field-tested in the reference (capacity controller). The
> minimum-billable-demand floor was decided by the owner 2026-08-02 (D16 pack A14,
> docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §3; it is
> reference-production-sourced against a real Belgian capacity-tariff bill, not stated in
> the thread.

### Requirement: Capacity budget denominated in the billed quantity

A demand budget SHALL be denominated in the billed, measured quantity — the meter's own
slot average, read as an unsigned import magnitude rather than under the signed grid
convention `energy-participants` _Provider roles_ fixes — with the planner booking against
a projection of the running slot average extrapolated to slot end, reconciled every cycle,
and shedding against `max(month-to-date billed peak, minimum billable demand)` less a
safety margin that is a site-configurable number of watts, shipped at the 300 W the
reference implementation runs.

#### Scenario: Projection reconciled while it still matters

- **GIVEN** a 15-minute slot four minutes in, whose running average extrapolates to
  5.1 kW against a month-to-date peak of 4.2 kW
- **WHEN** the cycle evaluates
- **THEN** the overshoot is acted on now rather than discovered at slot end, and the
  projection is recomputed from the meter on the next cycle rather than carried forward

#### Scenario: Estimates never become the billed number

- **GIVEN** commands that are envelopes, so booked estimates and metered draw diverge
  within the slot
- **WHEN** the slot closes
- **THEN** the committed peak is the metered slot average, and the next slot's projection
  starts from the meter — no estimate is ever committed as a billed quantity

#### Scenario: Shed margin is a stated configuration value

- **GIVEN** a site that has configured its capacity shed margin
- **WHEN** the shed threshold is computed
- **THEN** it is `max(month-to-date peak, minimum billable demand)` less exactly that
  margin, with the margin visible as configuration rather than compiled in

#### Scenario: A site that configures nothing still has a margin

- **GIVEN** a site that has never touched the capacity shed margin
- **WHEN** the shed threshold is computed
- **THEN** 300 W is subtracted, the shipped default being a starting value a site may
  change rather than a property of the tariff

#### Scenario: The free budget is the larger of the two, in both directions

- **GIVEN** a month-to-date peak of 4.2 kW against a minimum billable demand of 2.5 kW,
  and separately a month-to-date peak of 1.8 kW against the same 2.5 kW floor
- **WHEN** the shed threshold is computed for each
- **THEN** it is 4.2 kW less the margin in the first case and 2.5 kW less the margin in the
  second, agreeing with _Established peak as budget_ and with _Below the minimum billable
  demand there is nothing to shave_ rather than shedding below either

> Source: surfaced by the wave-1 prototype (E14 — see docs/PROTOTYPE_FEEDBACK.md), which
> found the corpus silent on whether a budget is denominated in estimates or in billed
> quantities and on how the error is reconciled; decided by the owner 2026-08-02 (D16 pack
> A14, with D17's Part B configurable shed margin — its shipped 300 W seeded from the
> reference's own `CAPACITY_SHED_MARGIN_W`, docs/OWNER_DECISIONS.md) — alternatives
> preserved in design.md §3 (reconcile at period end; a feedback loop inside the runtime
> floor). **Reference-production-sourced**: the tracker/shaving pair this describes runs
> against a real Belgian capacity-tariff bill. Not stated in the thread. The selector is
> `max` because the free budget is the **larger** of the two quantities; the decision pack
> transcribed the reference's `Math.min(mtdPeak, −minBillableW)`, in which **both terms are
> negative** because import is negative under the signed grid convention, so restating it in
> unsigned watts inverts the selector — which is why this requirement now says which
> quantity it is denominated in as well as how it is combined.

### Requirement: Metering slots follow the supplier's clock

Budget slots SHALL align to the supplier's metering intervals in wall-clock, zone-local
time — quarters starting at :00, :15, :30 and :45 for a 15-minute tariff — and never to a
rolling window over the trailing interval.

#### Scenario: Rollover at the supplier's boundary

- **GIVEN** a 15-minute capacity tariff in the site's zone
- **WHEN** the wall clock passes :15
- **THEN** the running average commits as that quarter's value and a fresh quarter starts,
  rather than a trailing 15-minute window sliding across the boundary

> Source: surfaced by the wave-1 prototype alongside E14; decided by the owner 2026-08-02
> (D16 pack A14, docs/OWNER_DECISIONS.md) — a rolling average never matches a bill, which
> is why the rolling-window alternative is recorded as rejected in design.md §3 rather than
> left as an implementation choice. Zone-local wall clock follows the corpus's convention of
> taking the site zone from core's `TimeZoneProvider`. Not stated in the thread.

### Requirement: Look-ahead and replanning live with the planner

This capability SHALL own look-ahead scheduling and the replanning of loads the runtime
floor deferred, consuming published deferral outcomes on its own cadence rather than
feeding a decision back into the cycle that produced them.

#### Scenario: A deferred load is replanned, not retried inside the cycle

- **GIVEN** the engine's runtime floor deferring a load it cannot admit and publishing
  that outcome with its reason
- **WHEN** the planner next runs
- **THEN** it schedules that load into a later window, and the cycle that deferred it
  stayed a pure function of its own snapshot

#### Scenario: One owner for replanning

- **GIVEN** the corpus's two candidate homes for replanning, the engine contract and this
  capability
- **WHEN** an implementer looks for the requirement that owns it
- **THEN** it is here, `define-engine-contract` carrying only the runtime floor and a
  terminal deferral-to-report path

> Source: surfaced by the wave-1 prototype (E16, L18 — see docs/PROTOTYPE_FEEDBACK.md),
> which found `fixtures/expected-boiler-control.csv` requires look-ahead rescheduling that
> no wave-1 requirement described, and left the ownership question open in both changes;
> decided by the owner 2026-08-02 (D16 pack A14, docs/OWNER_DECISIONS.md) — grid-constraints
> owns look-ahead and replanning, engine-contract owns the runtime floor and a
> deferral-to-report path only; alternatives preserved in design.md §4. Not stated in the
> thread.

### Requirement: Load balancing under a power budget

The system SHALL schedule consumers under a total power budget using the power figures
their class declares and their priority — a lower priority number being the better one —
so mutually exclusive loads are serialized onto the best slots instead of overlapping.

#### Scenario: Boiler and heater never together

- **GIVEN** a 3 kW boiler at priority 2 and 9 kW heating at priority 1 that both need
  hours under a 10 kW budget
- **WHEN** both are scheduled
- **THEN** the heating, whose lower priority number makes it the better-priority load,
  gets the best window and the boiler gets the next best non-overlapping one

#### Scenario: A borrowed power figure is reported

- **GIVEN** a Simple consumer that declares no `ratedPower`, so the budget falls back to
  its on-threshold
- **WHEN** the planner books it
- **THEN** the load is scheduled and the borrowed figure is reported as a declaration gap,
  never rejected

> Source: masipila's multi-objective sketch with maxPower + schedulingPriority
> ([1481931363](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931363));
> seime's limit-load-within-window need
> ([1481931351](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931351));
> priority direction decided by the owner 2026-08-02 (D4) and the per-class power figures by
> D15 (pack A6) — docs/OWNER_DECISIONS.md, alternatives preserved in design.md §5 and §1.
> D4 carries a known follow-on: the owner's live binding states the opposite for conflicts
> and needs reconciling (design.md §5). The fixture pair attributed to this requirement
> reproduces under a naive exclusion scheduler too (9 + 3 = 12 > 10), so passing it is
> necessary but not sufficient; a discriminating vector is to be **derived from the owner's
> site** (D19, task 1.3) and **does not exist yet**.

### Requirement: Region-configurable definitions

Constraint/fee definitions SHALL be configurable per region, date-dependent, and
updatable outside the core release cycle (e.g. as data files or add-ons), because
these rules change on regulatory timetables, not openHAB's.

#### Scenario: Tariff rule changes mid-year

- **WHEN** a grid operator changes its peak-fee formula effective a given date
- **THEN** the user updates the constraint definition without waiting for an openHAB
  release

> Source: seime's region/date-based downloadable configuration
> ([1481931351](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931351));
> mherwege's "engine needs flexibility to insert specific logic"
> ([1481931336](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931336)).
