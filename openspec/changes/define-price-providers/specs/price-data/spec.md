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

- **GIVEN** spot prices from an ENTSO-E source and the grid tariff from a separate
  calculator
- **WHEN** window optimization runs
- **THEN** it runs on the summed effective price

#### Scenario: Components sum below zero

- **GIVEN** a spot component more negative than the fixed components are positive
- **WHEN** the effective consumption price is composed
- **THEN** the negative effective price is carried through unclamped, because
  `Number:EnergyPrice` is a signed quantity and nothing in the plane assumes a price is
  non-negative

> Source: jlaur's elements model + "user should be able to configure which items to
> include" ([1482016387](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1482016387),
> [1489220814](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1489220814));
> Kai agreed ([1500300143](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1500300143));
> masipila's two-binding composition ([1492955803](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1492955803));
> zero and negative **consumption** prices — mentioned nowhere in the corpus until the
> wave-1 prototype raised L19 (see docs/PROTOTYPE_FEEDBACK.md) — decided by the owner
> 2026-08-02 (D17 · EL-13, docs/OWNER_DECISIONS.md) — alternatives preserved in
> design.md §4.

### Requirement: Generic grid-price provider

The system SHALL ship one generic, configurable grid-price provider that reads a future
price series from an Item and applies standard adjustments (VAT multiplier, unit
conversion such as EUR/MWh → ct/kWh, fixed fees, simple conditional tariffs), so common
cases need no custom binding.

#### Scenario: ENTSO-E raw to consumer price

- **GIVEN** a source publishing EUR/MWh without VAT
- **WHEN** the user configures ×VAT and /10
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

The system SHALL provide shared, reusable price calculations — at minimum the cheapest and
most expensive consecutive and non-consecutive selections for a request carrying a
`Duration`, over one costing function `cost(window, weights)` that defaults to flat
weights and takes a declared load curve as its weights when one exists — implemented once
and available to engines, rules and scripts, where a window starts only on a slot boundary
(a documented restriction of the search space, not a property of the optimum once weights
are non-flat), two slots are contiguous only when one ends exactly where the next begins,
and a request that cannot be met in full is answered with the best partial selection
carrying the requested and the granted duration rather than with silence.

#### Scenario: Cheapest consecutive window for a curve

- **GIVEN** a known load curve and a deadline of 07:00
- **WHEN** a rule asks for the cheapest consecutive window of 3 hours' duration before it
- **THEN** the shared calculation returns start time and expected cost, costing the curve
  as weights, without the rule reimplementing the math

#### Scenario: Energy under a curve

- **GIVEN** a relative-time load curve anchored at a candidate start, each of whose
  samples is the average power from its own timestamp to the next and whose last interval
  ends at the declared runtime
- **WHEN** the window is costed
- **THEN** each interval's energy is integrated LEFT-Riemann and priced at that interval's
  effective price, so two conforming implementations cost the same dishwasher identically

#### Scenario: A window starts on a slot boundary

- **GIVEN** a load curve whose cheapest placement would begin twenty minutes into an hourly
  slot
- **WHEN** the cheapest consecutive window is selected
- **THEN** the returned start is a slot boundary and not that cheaper offset start, the
  search space being restricted to boundaries by this requirement rather than the optimum
  happening to fall on one

#### Scenario: A gap breaks a consecutive window

- **GIVEN** a series with no entry covering 03:00 to 04:00
- **WHEN** a consecutive window of 2 hours is requested across that hole
- **THEN** no candidate spanning it is contiguous, because contiguity means each slot ends
  exactly where the next begins

#### Scenario: Not enough eligible slots

- **GIVEN** a 3-hour request and only 2 hours of eligible slots before the deadline
- **WHEN** the calculation runs
- **THEN** the best 2 hours come back carrying requested = 3 h and granted = 2 h, so a
  boiler that needs three hours and can get two gets two

#### Scenario: A day whose prices are all negative

- **GIVEN** every effective consumption price for the day below zero
- **WHEN** the cheapest window is selected
- **THEN** the signed prices rank unchanged and the most negative total wins, with nothing
  clamped at zero

> Source: the issue's founding requirement (calculations implemented once,
> [issue body](https://github.com/openhab/openhab-core/issues/3478)); masipila's
> contributed-algorithms concept ([1481931363](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931363)),
> whose production `GenericOptimizer` takes float hours and has never had a slot count;
> the request shape, contiguity, start granularity, shortfall answer and the weights
> parameter decided by the owner 2026-08-02 (D16 · pack A12, docs/OWNER_DECISIONS.md) —
> alternatives preserved in design.md §1, design.md §2 and design.md §3; the curve's time
> base and the named integration rule by the same decision (D16 · pack A13), whose own home
> is `add-named-profiles`; signed-price handling by D17 · EL-13 — alternatives preserved in
> design.md §4.

### Requirement: Feed-in pricing

The system SHALL model the feed-in (injection) price separately from the consumption
price, including zero and negative effective feed-in prices, so that a kilowatt-hour is
valued at the consumption price when imported and at the feed-in price when exported and
engines can weigh self-consumption against export honestly.

#### Scenario: Negative feed-in

- **WHEN** delivering to the grid costs more in tariffs than the spot revenue
- **THEN** the engine sees a negative feed-in price and prefers any local consumption

#### Scenario: One arithmetic, both directions

- **GIVEN** a positive consumption price and a negative feed-in price for the same slot
- **WHEN** the same kilowatt-hour is valued as imported and as exported
- **THEN** consuming it locally scores better under the cost objective and under the
  self-consumption objective alike, with neither objective needing a special case

> Source: jlaur's Danish net-tariff case + Kai's "we would need a way to express this"
> ([1482016387](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1482016387),
> [1482078918](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1482078918));
> the import/export valuation made definite alongside the owner's carbon decision of
> 2026-08-02 (D20, docs/OWNER_DECISIONS.md) — the cost side of that question was settled
> arithmetic and needed no decision, and it is the carbon term, not this one, that rests on
> the owner's reasoning (`define-optimization-objectives` _Carbon credit for exported
> energy_).

### Requirement: Delivery-day identity and the market zone

A price series SHALL carry the time zone of the market that produced it, so that a
delivery day is named by its own local date in that zone and is never inferred from the
site's zone or from UTC.

#### Scenario: The corpus fixture's own day

- **GIVEN** `fixtures/dayahead-prices.csv`, whose slots run 2023-03-23T23:00Z to
  2023-03-24T22:00Z
- **WHEN** the delivery day is named
- **THEN** it is the CET calendar day 2023-03-24 — neither the UTC day nor the publisher's
  own EET day

#### Scenario: A site in a different zone

- **GIVEN** a site whose zone differs from the market's
- **WHEN** the day's plan is derived and a seasonal parameter set is chosen for it
- **THEN** the day boundary follows the market zone carried on the series while the season
  boundary follows the site zone, and neither is inferred from the other

> Source: the corpus's own fixture is the evidence — `fixtures/dayahead-prices.csv` spans
> exactly the CET calendar day (see fixtures/README.md, "Slot geometry"); the silence was
> opened by the wave-1 prototype as I3 (see docs/PROTOTYPE_FEEDBACK.md), which found that a
> delivery day routinely spans two local dates with nothing saying which one names it;
> decided by the owner 2026-08-02 (D17 · EL-12c, docs/OWNER_DECISIONS.md) — alternatives
> preserved in `define-energy-levels` design.md §12.

### Requirement: Time resolution

All price and forecast handling SHALL be resolution-agnostic ("timeResolution", not
"granularity") and accept non-uniform intervals within one series, so a firm near-term
segment and a coarser far-term segment can share a single TimeSeries.

#### Scenario: Market switches to 15 minutes

- **WHEN** a source starts publishing 96 slots/day
- **THEN** composition and the shared window calculations keep working unchanged, their
  requests being durations rather than slot counts

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
> The scenario used to claim **level derivation** keeps working unchanged at 15 minutes too;
> it no longer does, because `define-energy-levels` _Level derivation_ configures its band
> counts in "hours" and the unit of those counts is still undecided
> (`define-energy-levels` design.md §7) — at 96 slots a day "4 blocked hours" is either 4
> slots or 16. The claim is narrowed to what this change can actually promise rather than
> left asserting something the corpus contradicts.
