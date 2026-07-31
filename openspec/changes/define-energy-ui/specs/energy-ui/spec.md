# energy-ui

## ADDED Requirements

### Requirement: Energy pages out of the box

The system SHALL provide MainUI energy pages automatically once the EMS is configured —
appearing without the user building any dashboard, and disappearing cleanly when the EMS
is removed.

#### Scenario: Fresh setup shows an overview

- **GIVEN** a configured EMS with declared participants
- **WHEN** the user opens MainUI
- **THEN** an energy section with a live overview exists without any page having been
  authored

#### Scenario: Clean removal

- **GIVEN** the provided energy pages
- **WHEN** the EMS is disabled or uninstalled
- **THEN** the pages disappear without orphaned UI configuration

> Source: Kai — "suitable widgets and whole pages for the Main UI … out of the box"
> ([1481931227](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931227));
> lsiepel's Dashboard/Insights module
> ([1481931215](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931215));
> feasibility **reference-sourced**: pages served via the core UI-component provider
> mechanism, installed/removed with the bundle, running in production.

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

> Source: follows from Kai's "out of the box" intent
> ([1481931227](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931227))
> and the propose-and-confirm flow of `discover-participants-from-model`; the explicit
> setup-surface requirement itself is **owner-directed (2026-07-31), not stated in the
> thread** — flagged like the discovery change.

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
they render from aggregated or pre-computed series at a bounded point count.

#### Scenario: A year in one fetch

- **GIVEN** a year of minute-level persistence
- **WHEN** the user opens the yearly energy view
- **THEN** it renders from on the order of one point per day, not from the raw minutes

> Source: **reference-production-sourced** (long-range charts over raw `everyChange`
> data degrade; a daily-rollup tier fixed it) — flagged as such; the calculation
> primitives (`riemannSum*`, `delta*`, …) exist in core persistence extensions today,
> the requirement is that the UI path uses bounded data.
