# energy-ui

## ADDED Requirements

### Requirement: Energy pages out of the box

The system SHALL provide MainUI energy pages automatically once the EMS is configured,
served by the framework through core's UI-component provider mechanism from a bundle
separate from the engine — appearing without the user building any dashboard, and
disappearing cleanly when that bundle is removed.

#### Scenario: Fresh setup shows an overview

- **GIVEN** a configured EMS with declared participants
- **WHEN** the user opens MainUI
- **THEN** an energy section with a live overview exists without any page having been
  authored

#### Scenario: Clean removal

- **GIVEN** the provided energy pages
- **WHEN** the EMS is disabled or uninstalled
- **THEN** the pages disappear without orphaned UI configuration

#### Scenario: The UI is separable from the engine

- **GIVEN** the pages served from their own bundle
- **WHEN** a future release moves the energy pages into MainUI itself
- **THEN** the change is the deletion of that bundle, the engine and its declarations
  being untouched

> Source: Kai — "suitable widgets and whole pages for the Main UI … out of the box"
> ([1481931227](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931227));
> lsiepel's Dashboard/Insights module
> ([1481931215](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931215));
> feasibility **reference-sourced**: pages served via the core UI-component provider
> mechanism, installed/removed with the bundle, running in production. The shipping path
> and the separate bundle were decided by the owner 2026-08-02 (D17, Part B UI-1,
> docs/OWNER_DECISIONS.md) — the MainUI-native path is preserved as the sequenced end state
> in design.md §1.

### Requirement: Overview minimum set

The energy overview SHALL show at least the site energy level, live per-participant power,
today's cost or self-consumption according to the active objective, the active grid
constraint's headroom, and one status line carrying degraded sources and declaration
errors.

#### Scenario: Everything needed to trust the engine, on one screen

- **GIVEN** a site with a capacity tariff, a selected objective and declared participants
- **WHEN** the user opens the overview
- **THEN** it shows the current site level, each participant's live power, today's figure
  for the active objective, the month-to-date peak as that tariff's headroom, and a status
  line

#### Scenario: Nothing wrong, nothing shouting

- **GIVEN** no degraded source and no declaration error
- **WHEN** the user opens the overview
- **THEN** the status line reports a healthy engine rather than disappearing, so its
  absence is never mistaken for silence

#### Scenario: A degraded source is visible without opening a log

- **GIVEN** a contributed forecast source that has gone dark and a participant whose
  declaration was skipped as malformed
- **WHEN** the user opens the overview
- **THEN** both are on the status line, sourced from the engine's status Item rather than
  from the UI's own inspection

> Source: this change's own candidate minimum set (design.md §2 — live flows, current and
> planned level, today's cost/self-consumption per the active objective, per-participant
> state), extended and frozen by the owner 2026-08-02 (D17, Part B UI-2,
> docs/OWNER_DECISIONS.md) with the grid-constraint headroom KPI (shipped in the reference
> implementation's `Now` tab) and the status line (where D16 pack A8's observability surface
> reaches a human) — alternatives preserved in design.md §2. mstormi's "UI time-sink"
> warning is respected by stating a floor, not a design: **not stated in the thread**.

### Requirement: Guided setup surface

The system SHALL provide a UI surface for declaring and editing participants and their
intent — demand, deadlines, priorities, level gates, objective selection — and for
confirming discovery proposals, so setup does not require hand-editing metadata or
configuration files.

#### Scenario: Boiler declared without textual config

- **GIVEN** a user who has never edited a `.items` file
- **WHEN** they declare their boiler through the setup surface
- **THEN** a participant declaration equivalent to a hand-written one is created

#### Scenario: Discovery proposals reviewed in place

- **GIVEN** pending proposals from semantic-model discovery
- **WHEN** the user reviews them in the setup surface
- **THEN** accepted proposals become declarations and rejected ones are discarded

#### Scenario: A malformed declaration is visible where it was made

- **GIVEN** a declaration whose protection parameter is mistyped, so the whole participant
  is skipped
- **WHEN** the user opens the setup surface for that item
- **THEN** the error is shown against the parameter that caused it, and the device is
  visibly unmanaged rather than silently so

> Source: follows from Kai's "out of the box" intent
> ([1481931227](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931227))
> and the propose-and-confirm flow of `discover-participants-from-model`; the explicit
> setup-surface requirement itself is **owner-directed (2026-07-31), not stated in the
> thread** — flagged like the discovery change. The declaration-error scenario follows the
> owner's 2026-08-02 decisions (D16 pack A8 and D18 Part C EP-5.1,
> docs/OWNER_DECISIONS.md — A8: skip the whole participant, report through a
> `ConfigStatusProvider` keyed on the item; EP-5.1: the `energy:` schema is
> published as a config description, which is what lets a UI render this form and this error
> without EMS-specific UI code); alternatives preserved in
> `define-extension-points` design.md §5.5.

### Requirement: Driven by declared participants

The energy pages SHALL derive their content from the declared participants — a
participant added, changed or removed is reflected without page editing.

#### Scenario: New consumer appears

- **GIVEN** the energy overview is open
- **WHEN** the user declares a new consumer
- **THEN** it appears on the pages without anyone editing a page definition

> Source: follows from item-level wiring in Kai's design
> ([1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374));
> live participant-driven rebuild **reference-sourced** (metadata-change listener in
> production).

### Requirement: Past and future in one view

The insights view SHALL chart recorded actuals and future series (forecasts, planned
levels, planned schedules) on one timeline, so the user sees what happened and what the
EMS intends next.

#### Scenario: Today with forecast ahead

- **GIVEN** persisted actuals and a solar forecast series
- **WHEN** the user views today
- **THEN** actuals render up to now and the forecast continues ahead of now on the same
  axis

> Source: lsiepel — "an area where it can supply charts / insights"
> ([1481931215](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931215));
> future-timestamped series are first-class (`forecast-data`); forecast-ahead-of-now
> charting proven in the reference.

### Requirement: Long-range views stay fast

Month- and year-scale views SHALL NOT require fetching every raw persisted datapoint;
they render from pre-computed rollup series — one point per day, written into dedicated
items, snapshotted before the day boundary with each increment derived from the last
reading before the reset.

#### Scenario: A year in one fetch

- **GIVEN** a year of minute-level persistence
- **WHEN** the user opens the yearly energy view
- **THEN** it renders from on the order of one point per day, not from the raw minutes

#### Scenario: The day boundary does not eat a day

- **GIVEN** a counter that resets at midnight
- **WHEN** the daily rollup point is written
- **THEN** it is snapshotted before the boundary and its increment is derived from the last
  reading before the reset, so neither the last minutes of the day nor the reset itself
  distorts the series

> Source: **reference-production-sourced** (long-range charts over raw `everyChange`
> data degrade; a daily-rollup tier fixed it) — flagged as such; the calculation
> primitives (`riemannSum*`, `delta*`, …) exist in core persistence extensions today,
> the requirement is that the UI path uses bounded data. The mechanism was decided by the
> owner 2026-08-02 (D18, Part C UI-3, docs/OWNER_DECISIONS.md): pre-computed rollups,
> because `FilterCriteria` carries no aggregation field, so the alternatives would change an
> interface every persistence add-on implements — those alternatives are preserved in
> design.md §3.
