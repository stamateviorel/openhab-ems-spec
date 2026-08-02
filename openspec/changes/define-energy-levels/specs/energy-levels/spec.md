# energy-levels

## ADDED Requirements

### Requirement: Four-level scale

The system SHALL express energy availability as an ordered four-level scale —
blocked / normal / encouraged ("low price") / overcapacity — whose numeric encoding is
fixed centrally as `blocked = 0`, `normal = 1`, `encouraged = 2`, `overcapacity = 3` and
used identically by every component that exchanges a level as a number, so that a larger
code always means more available energy and no exchange has to agree an encoding of its
own.

#### Scenario: Numeric exchange

- **GIVEN** a level written as a number by one component — a published Item state, a
  planned series, an acceptance fixture — and read back by a second, independently written
  one
- **WHEN** each maps between the level names and the fixed codes
- **THEN** the round trip returns the level it started from, for each of the four levels

#### Scenario: The fixture's codes are the central ones

- **GIVEN** `fixtures/expected-planned-levels.csv`, whose cheapest slots carry `3` and
  whose most expensive carry `0`
- **WHEN** a conforming classifier reproduces that file
- **THEN** it does so under the centrally fixed encoding rather than under one chosen for
  that file, and publishes the same codes on every other surface

#### Scenario: Binary collapse

- **WHEN** a Simple ON/OFF consumer subscribes to levels
- **THEN** the four levels collapse to allow/deny for it

> Source: mstormi's Energieniveau model ([1481931249](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931249),
> [docs](https://storm.house/docs/#Energieniveaumanagement)), masipila's SG-mode framing
> ([1481931320](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931320)),
> EVCC mode equivalence ([1481931278](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931278));
> sharpened after the wave-1 prototype surfaced L1 (see docs/PROTOTYPE_FEEDBACK.md); the
> encoding decided by the owner 2026-08-02 (D12, docs/OWNER_DECISIONS.md) — alternatives
> preserved in design.md §2. The level **names** were not decided and stay open in
> design.md §1.

### Requirement: SG-ready mode mapping

The system SHALL map the four levels onto the four SG-ready modes by a named
correspondence — blocked ↔ mode 1, normal ↔ mode 2, encouraged ↔ mode 3,
overcapacity ↔ mode 4 — and never by arithmetic on the level code, because SG-ready
numbers its modes 1–4 while the levels are encoded 0–3 and the two therefore differ by
exactly one throughout, with the SG-Ready obligation to limit mode 1 remaining a
participant-side obligation that core does not enforce.

#### Scenario: Overcapacity drives mode 4

- **GIVEN** a site level of overcapacity, whose code is `3`
- **WHEN** an SG-ready heat pump participant is driven from it
- **THEN** mode 4 is asserted — never mode 3 — and no translation logic is needed in user
  rules

#### Scenario: Mode round trip

- **GIVEN** each of the four levels in turn
- **WHEN** it is mapped to an SG-ready mode and back
- **THEN** the level returns unchanged, and no level's code equals the mode it maps to

> Source: masipila's SG-mode framing ([1481931320](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931320))
> and mstormi's SG-ready device model ([1481931249](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931249));
> split out of _Four-level scale_ and made explicit after the wave-1 prototype surfaced L2
> (see docs/PROTOTYPE_FEEDBACK.md), which found a reader of the fixture alone would publish
> 0–3 straight onto an SG-ready channel; decided by the owner 2026-08-02 (D12 for the
> offset, D17 · EL-3c for the mode-1 cap's scope; docs/OWNER_DECISIONS.md) — alternatives
> preserved in design.md §3, together with the two limits the decision pack fact-checked
> against the BWP SG-Ready specification and the one the participant model cannot express.

### Requirement: Level derivation

The system SHALL derive the planned level for each slot from the price schedule as a rank
within the planning horizon — never as a claim about the absolute price — with the number
of hours assigned to each non-normal level user-configurable, including via rules so that
e.g. cold weather can reduce blocked hours, and with the classification of slots that tie
on price settled by a stated tie-break rather than by the order in which slots are
evaluated.

#### Scenario: Cheapest-hours classification

- **GIVEN** a user configuration of 4 overcapacity, 4 low-price and 4 blocked hours
- **WHEN** tomorrow's 24 prices arrive
- **THEN** the planned levels mark the 4 cheapest hours overcapacity, the next 4
  low-price, the 4 most expensive blocked, and the rest normal

#### Scenario: Repeated prices

- **GIVEN** more slots tied on the same price than the band being filled has room for
- **WHEN** the plan is derived twice from that same series
- **THEN** the same slots land in the band both times, decided by the stated tie-break
  and not by which slot happened to be visited first

#### Scenario: A day whose prices are all negative

- **GIVEN** a delivery day on which every effective consumption price is below zero
- **WHEN** the plan is derived
- **THEN** the dearest slots are still blocked and the cheapest still overcapacity,
  because a level ranks slots against each other and makes no claim about the sign or size
  of the price

> Source: mstormi's base-plus-escalation with user-configurable hours ("that's key to
> applicability", [1481931249](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931249)),
> masipila's algorithm sketch incl. rule-updatable hour counts
> ([1481931363](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931363));
> sharpened after the wave-1 prototype surfaced L3 and L19 (see docs/PROTOTYPE_FEEDBACK.md);
> the rank-not-absolute reading decided by the owner 2026-08-02 (D17 · EL-13,
> docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §13. The tie-break itself
> was not decided and stays open in design.md §6, so the _Repeated prices_ scenario is not
> runnable until a maintainer names one (pack A11's "price ascending, then slot start
> ascending" is one comparator, preserved there). The **unit of the configured counts** —
> hours or slots — is likewise undecided (design.md §7), so this requirement is fully
> definite only on an hourly series: on 15-minute data "4 blocked hours" is either 4 slots
> or 16, and nothing states which. Live escalation moved to _Surplus escalation of the
> current level_.

### Requirement: Surplus escalation of the current level

The current level SHALL escalate above its planned value in graded steps as site surplus
rises — reaching encouraged at a site-configured `encouragedFrom` threshold in watts, which
has no shipped default, so the current level does not escalate above its planned value
until a site sets one, and overcapacity at `overcapacityFrom`, which defaults to twice
`encouragedFrom` — where surplus is the site's grid export plus the battery charging the
engine can reclaim, read under the corpus's single sign convention (grid + = export,
battery + = charging, PV + = producing, consumers + = consuming).

#### Scenario: First graded step

- **GIVEN** an hour whose planned level is normal, and `encouragedFrom` = 1500 W
- **WHEN** live surplus reaches 1500 W
- **THEN** the current level reads encouraged while the stored plan still reads normal

#### Scenario: Second graded step

- **GIVEN** the same configuration, so `overcapacityFrom` defaults to 3000 W
- **WHEN** live surplus reaches 3000 W
- **THEN** the current level reads overcapacity

#### Scenario: Battery charging is part of the surplus

- **GIVEN** nothing being exported and 3 kW charging a house battery that the engine may
  reclaim for a better-priority consumer — one whose priority number is lower than the
  battery's
- **WHEN** escalation is evaluated
- **THEN** that 3 kW counts towards the thresholds exactly as exported power would, so a
  solar-first consumer is not starved by the battery absorbing everything

#### Scenario: A site that has set no threshold does not escalate

- **GIVEN** a fresh site that has never configured `encouragedFrom`
- **WHEN** surplus rises to any value
- **THEN** the current level stays at its planned value and the absence of a threshold is
  what says so, rather than escalation happening at a number nobody chose

> Source: mstormi's base-plus-escalation ([1481931249](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931249))
> and storm.house's escalation on a configured watt threshold; sharpened after the wave-1
> prototype surfaced L7 and L8 and shipped the requirement inert by defaulting escalation
> to `none` (see docs/PROTOTYPE_FEEDBACK.md); decided by the owner 2026-08-02 (D17 · EL-8a
> for the shape and thresholds, D10 for what counts as surplus, D11 for the sign
> convention; docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §8, in
> `define-engine-contract` design.md §11 (surplus) and in
> `define-participant-model` design.md §5.11 (signs). Still open, by the owner's note: whether
> the surplus figure is instantaneous, averaged or forecast, and whether an already-running
> managed consumer's own draw counts towards it. **`encouragedFrom` has no shipped value and
> none was invented**: EL-8a gave the graded shape and the 2× relation between the two
> thresholds, never a base number, and the reference implementation has no site-level
> escalation threshold to read one off — its 2000 W surplus on-threshold is a **per-consumer**
> switching parameter, a different quantity. The requirement therefore states what an unset
> threshold means instead of guessing what it should be; supplying a default is a one-line
> change for whoever can source one.

### Requirement: The current level is a product of the engine's cycle

The current level SHALL be computed by the engine from the same one-cycle snapshot it
reasons about, by calling level derivation as a pure function of that snapshot, so that
the level and the readings it was derived from can never disagree inside one cycle —
publication of the current level as an Item and of the plan as a TimeSeries being an
output of that computation and never the path by which the engine obtains a level.

#### Scenario: One reading, one level

- **GIVEN** a cycle whose snapshot holds a single grid reading and a single surplus figure
- **WHEN** the level is derived and the same cycle allocates power to consumers
- **THEN** both use that one reading, and no second reading is taken to answer "what is
  the level"

#### Scenario: Shadow mode still has a level

- **GIVEN** the engine in shadow mode, where nothing is written to any Item
- **WHEN** a cycle runs
- **THEN** it still computes and logs a current level, because the level does not arrive
  by reading back a published Item

> Source: opened by the wave-1 prototype as I1 (see docs/PROTOTYPE_FEEDBACK.md), which
> resolved the recursion between this change and `define-engine-contract` in order to
> compile and reported it as a contradiction rather than an answer, since the corpus never
> said which way the dependency ran; decided by the owner 2026-08-02 (D6,
> docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §15, in design.md §14 and,
> on the engine's side, in `define-engine-contract` design.md §4 and
> `define-engine-contract` design.md §18.

### Requirement: Planned schedule vs. current level

The system SHALL keep the planned level schedule (a future-timestamped TimeSeries) and
the current level (an Item) as separate artifacts — both of them reports of what the
engine computed, not inputs to it — where live conditions may override the plan for the
current slot.

#### Scenario: Plan says normal, sun says overcapacity

- **GIVEN** a planned series holding "normal" for the current hour
- **WHEN** PV excess is live
- **THEN** the current-level Item reads overcapacity while the stored plan stays intact

> Source: masipila's planned-mode / current-mode decoupling
> ([1481931320](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931320),
> [1481931363](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931363));
> the artifacts' direction (reports, not inputs) decided by the owner 2026-08-02 (D6,
> docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §15. Whether the plan
> rides on the same Item as the current level or on a second one was not decided and stays
> open in design.md §14.

### Requirement: Level when no plan covers the present instant

The current level SHALL read normal whenever no plan covers the present instant — before
the first prices arrive, after the plan's last slot, or inside a gap in a non-uniform
series — with the absence of the plan reported separately, so that a missing price feed is
visible rather than indistinguishable from a genuinely normal hour.

#### Scenario: No prices have ever arrived

- **GIVEN** a fresh site whose price source has never delivered a series
- **WHEN** the engine evaluates
- **THEN** the current level reads normal and the plan is reported absent

#### Scenario: The plan runs out

- **GIVEN** a plan whose last slot ends at 23:00
- **WHEN** the clock passes 23:00 with no newer series published
- **THEN** the current level falls back to normal and the absence is reported, rather than
  the last planned level being held indefinitely

> Source: the gap was opened by the wave-1 prototype as L9 (see docs/PROTOTYPE_FEEDBACK.md),
> which made it a configuration parameter defaulting to normal and recorded that as a build
> artefact rather than a proposal; decided by the owner 2026-08-02 (D17 · EL-9,
> docs/OWNER_DECISIONS.md), grounded in the reference implementation, whose
> `pricePercentileLevel` returns the normal level for a missing schedule in production —
> alternatives preserved in design.md §9, including the most-restrictive reading that turns
> a failed price fetch into a cold house.

### Requirement: Re-derivation when the configuration changes

A change to the configured hour counts SHALL take effect immediately, re-deriving the
plan over the whole published series and re-publishing `[now, end)` as one TimeSeries with
`Policy.REPLACE` — whose bounds core converts in the system default zone — while leaving
already-elapsed entries untouched.

#### Scenario: A cold-weather rule reduces the blocked hours at midday

- **GIVEN** a plan published this morning for the rest of the day
- **WHEN** a cold-weather rule lowers the blocked-hour count at 14:00
- **THEN** the entries from 14:00 to the plan's end are replaced in one series, and this
  morning's already-elapsed entries stay exactly as they were

#### Scenario: A fresh plan replaces an old one

- **GIVEN** a plan already published for tomorrow
- **WHEN** a corrected price series arrives for the same day
- **THEN** the re-derived plan replaces the old entries for that range rather than being
  merged into them

> Source: masipila's rule-updatable hour counts
> ([1481931363](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931363))
> and mstormi's cold-weather example ([1481931249](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931249)),
> whose timing the prototype found unstated as L22 (see docs/PROTOTYPE_FEEDBACK.md);
> decided by the owner 2026-08-02 (D17 · EL-10, docs/OWNER_DECISIONS.md), on core's own
> `Policy.REPLACE`, which deletes exactly the published range — alternatives preserved in
> design.md §10.

### Requirement: Window strategies

The level classifier SHALL select windows from a request carrying a `Duration`, in both
non-consecutive form (the cheapest slots totalling that duration) and consecutive form
(one uninterrupted stretch covering it) and independent of the market time resolution
(60- or 15-minute slots), evaluated through the same shared calculation the price plane
specifies (`define-price-providers` _Shared window calculations_) so that contiguity,
start granularity and shortfall behave identically everywhere, with a slot-count entry
point retained for the "N cheapest slots" case whose candidates are compared on
duration-weighted mean price.

#### Scenario: Consecutive window at 15-minute resolution

- **GIVEN** a consumer that needs 2 uninterrupted hours
- **WHEN** prices arrive in 15-minute slots
- **THEN** the classifier finds the cheapest consecutive window covering that duration —
  8 slots at this resolution

#### Scenario: Not enough eligible slots

- **GIVEN** a 3-hour request and only 2 hours of eligible slots before the deadline
- **WHEN** selection runs
- **THEN** the best 2 hours are returned, carrying requested = 3 h and granted = 2 h,
  rather than nothing

> Source: masipila's consecutive vs. non-consecutive discussion and 15-minute market
> outlook ([1481931313](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931313),
> [1481931320](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931320));
> the five silences the prototype had to guess at (L13–L17, see docs/PROTOTYPE_FEEDBACK.md)
> decided by the owner 2026-08-02 (D16 · pack A12, docs/OWNER_DECISIONS.md) — alternatives
> preserved in design.md §11.

### Requirement: Seasonal window defaults

Level derivation SHALL select its parameter set from user-editable fixed month-day ranges
evaluated in the site's own time zone (core's `TimeZoneProvider`), shipping summer
15 May – 15 September, winter 1 November – 20 March and transition for the rest of the
year, as production experience shows one static configuration does not fit the year.

#### Scenario: Winter widens cheap windows

- **GIVEN** seasonal mode is on
- **WHEN** the date enters winter on 1 November in the site's zone
- **THEN** the configured winter hour-counts replace the transition ones without user
  action

#### Scenario: A market in another zone does not move the season

- **GIVEN** a site whose zone differs from the market's
- **WHEN** the season boundary passes
- **THEN** it passes at the site's midnight, because a season is a property of the
  building, while the delivery day being planned is still named in the market's zone

> Source: seasonal profiles raised by mstormi ([5016228379](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5016228379));
> percentile+seasonal windows field-tested in the reference
> ([EMS_ENGINE](https://github.com/stamateviorel/openhab-binding-emsmanager/blob/main/docs/EMS_ENGINE.md));
> the missing grammar was opened by the prototype as L20/L21/I3 (see
> docs/PROTOTYPE_FEEDBACK.md); boundaries, shipped dates and zone decided by the owner
> 2026-08-02 (D17 · EL-12a and EL-12b, docs/OWNER_DECISIONS.md), the dates being
> storm.house's own — alternatives preserved in design.md §12. The delivery-day half of
> that section is stated in `define-price-providers` _Delivery-day identity and the market
> zone_.
