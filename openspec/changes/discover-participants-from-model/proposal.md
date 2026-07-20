# Discover participants from the semantic model

**Wave 1 (onboarding)** — builds on [`define-participant-model`](../define-participant-model/).
An assisted-setup layer: propose the participant set from what openHAB already knows.

> **Provenance note.** Unlike the other changes here, whose requirements each trace to a
> specific #3478 comment, this one is a **fresh design proposal (owner + assistant,
> 2026-07-20), not yet raised in the thread.** It rests on two solid things: openHAB's
> existing **semantic model** (verifiable — the tags below are from
> `openhab-core` `SemanticTags.csv`) and Kai's 2023 intent to *mark* energy items via
> metadata and get energy UI "out of the box". Flagged so a reviewer treats it as a
> proposal to react to, not thread-vetted consensus.

## Why

The participant model asks the user to declare each provider/consumer via `energy:`
metadata. But a semantically modelled openHAB instance **already classifies most of
it**: Equipment tags `Battery` / `Inverter` / `SolarPanel` / `EVSE` / `ElectricMeter`
(under `PowerSupply`/`Sensor`), `HeatPump` / `Boiler` (under `HVAC`), `Dishwasher` /
`WashingMachine` / `Freezer` (under `WhiteGood`); Point types `Measurement` / `Setpoint`
/ `Switch` / `Control` / `Forecast`; Properties `Power` / `Energy` / `Current` /
`Voltage`. That vocabulary maps almost 1:1 onto the participant model's roles and four
consumer classes. So the EMS can **propose a near-complete setup** and let the user
confirm only what the model cannot hold. This is the recurring "energy dashboard out of
the box" wish in #3478.

## What changes

- Define a `participant-discovery` capability: how the semantic model maps to participant
  roles/classes, that discovery **proposes** (never silently activates) declarations,
  what stays user-owned, and how it coexists with explicit `energy:` declarations.

## Non-goals

- Replacing explicit declaration — discovery *seeds* `energy:` tags; both coexist and the
  explicit one wins.
- Inferring intent/sign/protections — those are not in the model (see the learning layer,
  wave 4, and confirm-at-setup).
- Binding- or name-heuristic discovery — this is semantic-model driven only.

## Impact

- Additive onboarding layer over the participant model; no change to the core model.
