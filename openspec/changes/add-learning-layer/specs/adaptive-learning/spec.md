# adaptive-learning

## ADDED Requirements

### Requirement: Learned thermal model
The system SHALL be able to learn a building thermal model (heat-loss rate / thermal
mass) from persisted indoor temperature, weather and heating-power data, expose its
quality (error metric), and use it to derive settings such as maximum heating-off
duration and pre-heat lead time.

#### Scenario: Max-off from the model
- **WHEN** the learned model has converged within its error bound
- **THEN** the engine can compute how long heating may stay off before the configured
  comfort floor is reached, instead of using a hand-tuned constant

> Source: mstormi's learning-layer vision ([5016228379](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5016228379));
> masipila's two-winters experience ([5017073428](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5017073428));
> prior art: online-learned RC model + pre-conditioning planner in the reference
> ([repo](https://github.com/stamateviorel/openhab-binding-emsmanager)).

### Requirement: Learned load profiles
The system SHALL be able to derive a device's load-curve profile from persisted runs
(automatically mapping a program from its recorded consumption) and publish it as a
named profile, updating it as new runs are recorded.

#### Scenario: Program mapped from history
- **WHEN** enough runs of the "60 °C cottons" program are persisted
- **THEN** a named profile with the measured curve becomes available for scheduling,
  replacing the flat-consumption assumption

#### Scenario: SG-ready device with richer telemetry
- **GIVEN** a heat pump controlled only through SG-ready modes but exposing live power or
  metrics over Modbus or a cloud interface
- **WHEN** that telemetry is persisted
- **THEN** the learning layer refines the device's consumption prediction from it, even
  though the control surface carries no power setpoint

> Source: jlaur's founding idea ([1481931141](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931141));
> mstormi's "adapt/replace the consumption TimeSeries profile from recorded historic
> data" ([5016228379](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5016228379))
> and SG-ready-but-measured dynamic consumers
> ([5020830338](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5020830338));
> normalized-pattern observation (lsiepel,
> [5003008753](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5003008753)).

### Requirement: Propose, don't override
Learned values SHALL be applied as reviewable proposals or clearly-flagged overrides —
never silently replacing user configuration, with safety-relevant limits always
remaining user-owned.

#### Scenario: Model drifts
- **GIVEN** a previously converged model
- **WHEN** its error metric degrades (sensor moved, renovation)
- **THEN** the system falls back to the user-configured values and surfaces the change,
  rather than acting on a bad model

> Source: guardrail synthesized from the thread's shadow-first validation culture
> ([5014707411](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5014707411));
> marked for explicit review as the only requirement here not stated verbatim by a
> thread participant.

### Requirement: On-premise learning
All learning SHALL run on locally persisted data without requiring cloud services.

#### Scenario: Offline site
- **WHEN** the installation has no internet beyond its forecast sources
- **THEN** thermal-model and profile learning still function on local persistence

> Source: openHAB's local-first principle; all named prior art (storm.house,
> spot-price-optimizer, reference) learns/operates on local data.
