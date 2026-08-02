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

#### Scenario: A consumer that names no profile still has one

- **GIVEN** a consumer declared without any named profile, as wave 1 declares them
- **WHEN** the engine plans it
- **THEN** it is treated as carrying exactly one implicit, unnamed active profile — whose
  curve is the rectangular rated-power-over-runtime default — so wave 1 and wave 3 are one
  model rather than two

> Source: mstormi ([5016228379](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5016228379)),
> seconded from a second installation by masipila
> ([5017073428](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5017073428)),
> program selection in jlaur's founding comment
> ([1481931141](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931141));
> the always-exactly-one-active-profile reading, including the implicit wave-1 profile, was
> decided by the owner 2026-08-02 (D16 pack A13, docs/OWNER_DECISIONS.md) — alternatives
> preserved in design.md §1.

### Requirement: Profile contents

Each named profile SHALL be able to carry a load curve as a relative-time shape — samples
offset from the profile's start, each the average power over the interval to the next
sample, the last interval ending at the declared runtime, integrated LEFT-Riemann, scaled
by a scale factor, and capped at 1440 samples as a declaration guard that carries no
semantic meaning — together with the class parameters it overrides (demand, deadline,
protections), so the same representation covers consumer programs and seasonal producer
curves.

#### Scenario: Washing program curve

- **GIVEN** the `fullsize` profile carrying its measured curve (peak 1:30 in, one hour
  long) and a scale factor
- **WHEN** a window is costed for it
- **THEN** the cost is computed over that curve, not over a flat average

#### Scenario: Range and intervals suit the device

- **GIVEN** a white-goods profile spanning a day at 1-minute intervals and a heat-pump
  profile spanning a year with mixed intervals
- **WHEN** each profile is scheduled
- **THEN** both are carried in one variable-interval representation sized to the device's
  cycle, each sample carrying its own spacing rather than inheriting an implied one

#### Scenario: The shape is anchored when it is scheduled

- **GIVEN** a stored profile whose samples are offsets from start, denoting no absolute
  time at rest
- **WHEN** the planner schedules it at a candidate start
- **THEN** it becomes a `TimeSeries` anchored at that start, and the same shape scheduled
  at a different start yields a different series without the profile changing

#### Scenario: The sample cap is a guard, not a resolution

- **GIVEN** a profile declaring more than 1440 samples
- **WHEN** the declaration is read
- **THEN** it is refused as a malformed declaration, and a profile at or under the cap is
  accepted whatever its spacing, the number bounding what a declaration may assert rather
  than saying anything about how finely a curve should be described

#### Scenario: Two implementations cost one curve identically

- **GIVEN** a curve whose samples are unevenly spaced
- **WHEN** two conforming implementations compute the energy of a window over it
- **THEN** both use LEFT-Riemann integration over each sample's own interval and get the
  same answer

> Source: "The TimeSeries and the scale factor should be part of the profile"
> ([5016228379](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5016228379));
> range/interval flexibility, mixed intervals in one series (white goods a day, car a
> week or two, heat pump a year) from mstormi
> ([5019271702](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5019271702));
> per-level PowerProfiles (lsiepel,
> [1481931303](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931303));
> curve shape precedent: jlaur's mapped dishwasher program
> ([1481931141](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931141));
> relative-time storage, per-sample interval semantics and LEFT-Riemann integration decided
> by the owner 2026-08-02 (D16 pack A13, docs/OWNER_DECISIONS.md) — this **replaces** the
> earlier "as a TimeSeries (of any range and interval spacing)" wording, whose absolute-time
> reading is preserved as the alternative in design.md §3 along with the cost of the change
> (a relative-time profile does not ride the price/forecast storage plane). Sample semantics
> and the 1440-sample declaration guard are prototype-surfaced (A8), not stated in the
> thread.

### Requirement: Profiles are updatable at any time

A profile's curve SHALL be read-write and updatable at any time, so it can be refined or
replaced live — by the learning layer or by richer telemetry — without redefining the
profile.

#### Scenario: Learned curve overwrites the baseline

- **GIVEN** a profile carrying an initial curve
- **WHEN** the learning layer produces a better curve from recorded runs
- **THEN** the profile's curve is overwritten in place and the next plan uses it

> Source: mstormi ([5020830338](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5020830338));
> ties to the learning layer (wave 4). "TimeSeries" reworded to "curve" following the
> owner's 2026-08-02 decision that a stored profile is a relative-time shape (D16 pack A13,
> docs/OWNER_DECISIONS.md) — alternative preserved in design.md §3; the read-write
> requirement itself is unchanged. A learned curve is applied under the learning layer's
> propose-don't-override rule (`adaptive-learning`).

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

### Requirement: Hands-off outranks profile selection

A consumer the user marked `never` SHALL stay hands-off whichever profile is active, no
named profile being able to carry, clear or weaken that flag.

#### Scenario: A seasonal switch does not resume a hands-off device

- **GIVEN** an EV charger marked `never` by its owner and a profile switch from `winter`
  to `summer`
- **WHEN** the new profile becomes active
- **THEN** the charger stays hands-off, and the engine writes nothing to it

#### Scenario: A profile still parameterizes what the user owns

- **GIVEN** a `summer` profile that shortens a heat pump's maximum-off time
- **WHEN** it becomes active on a consumer not marked `never`
- **THEN** the protection parameter changes as declared, because a profile is the user's
  own configuration — the flag it cannot touch is the hands-off prohibition

> Source: decided by the owner 2026-08-02 (D13, docs/OWNER_DECISIONS.md), whose
> engine-owned prohibition set lifts `never` off the Simple class onto the consumer so it
> applies to all four classes — which creates this question, because a named profile is "a
> complete parameterization of the participant's class". Alternatives preserved in
> design.md §4 (making `never` a per-profile parameter, which would express "hands off in summer" at
> the cost of a prohibition a schedule can switch off). The consumer-side requirement lives
> in `define-participant-model`; the contributor-side boundary in `extension-surface`
> _Engine-owned prohibitions are closed to contributors_. Not stated in the thread.
