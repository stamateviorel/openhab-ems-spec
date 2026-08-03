# optimization-objectives

## ADDED Requirements

### Requirement: Selectable objective

The system SHALL let the user select the optimization objective — at minimum lowest
cost, maximum self-consumption, and lowest carbon — instead of assuming lowest cost, the
surplus that the self-consumption objective optimizes into being `max(0, grid +
reclaimable)`: the site's signed grid reading plus the battery charging the engine can
reclaim, netted against any import and never negative, read under the corpus's single sign
convention.

#### Scenario: Self-consumption over cheap grid

- **GIVEN** the self-consumption objective and a live PV surplus during a cheap grid hour
- **WHEN** a deferrable load is planned
- **THEN** it is placed into the surplus rather than into the cheap grid hour

#### Scenario: Battery charging is reclaimable surplus

- **GIVEN** the self-consumption objective, nothing being exported, and 3 kW charging the
  house battery
- **WHEN** a deferrable load whose priority number is lower than the battery's — the
  better priority — is planned
- **THEN** it is placed now against that reclaimable charge rather than waiting for grid
  export to appear, because battery charging is a decision the engine itself made

#### Scenario: Reclaimable charge under an importing meter

- **GIVEN** the same objective, the site importing 1 kW, and 3 kW charging the battery
- **WHEN** the same load is sized against the surplus
- **THEN** 2 kW is available to it, the import being netted off first, so placing the load
  does not deepen the import the objective exists to avoid

#### Scenario: Carbon over price

- **GIVEN** the carbon objective and a green-share series showing a very renewable
  afternoon that is slightly pricier than the night
- **WHEN** the same load is planned
- **THEN** it is placed into the greener afternoon

> Source: lsiepel — "look for cheapest or for max self provided or whatever user defined
> preferences" ([1481931215](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931215));
> jlaur — "prioritize Planet Earth over price"
> ([1482016387](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1482016387));
> the word "surplus", which the wave-1 prototype found undefined anywhere in the corpus
> (E7, see docs/PROTOTYPE_FEEDBACK.md) even though this scenario places a load into it,
> decided by the owner 2026-08-02 (D10, with the sign convention D11;
> docs/OWNER_DECISIONS.md) — alternatives preserved in
> `define-engine-contract` design.md §11 and in
> `define-participant-model` design.md §5.11. The `max(0, …)` composition **amends D10 as
> literally worded** and comes from the owner's follow-up 2026-08-03 (D27,
> docs/OWNER_DECISIONS.md), which nets the reclaimable charge against import before it is
> offered to a load — alternatives preserved in `define-energy-levels` design.md §8 and in
> `define-engine-contract` design.md §11. Still open by the owner's
> own note on D10, and left open by the wording above: whether that figure is instantaneous,
> averaged or forecast, and whether an already-running managed consumer's own draw counts.

### Requirement: Carbon data as a first-class series

The system SHALL consume CO₂-intensity or green-share data as future-timestamped series
through the same data plane as prices and forecasts, from pluggable sources.

#### Scenario: Green-share series drives ranking

- **GIVEN** a contributed source publishing tomorrow's renewable share per hour
- **WHEN** the carbon objective ranks hours
- **THEN** the ranking uses that series exactly as the cost objective uses the price
  series

> Source: jlaur — CO₂/green datasets exist and "do not necessarily map directly to the
> prices" ([1482016387](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1482016387));
> storage model shared with `price-data`/`forecast-data`; emissions-provider prior art in
> the reference (fixed factors / live grid-intensity source).

### Requirement: Carbon credit for exported energy

The carbon objective SHALL credit exported energy with the generation it displaces only
while the effective feed-in price is zero or above, treating export at a negative feed-in
price as earning no displacement credit — the reasoning being that a negative price is the
market refusing the power, so the realistic outcome is curtailment and the displacement
the credit stands for never happens.

#### Scenario: Negative feed-in withdraws the credit

- **GIVEN** the carbon objective and an hour whose effective feed-in price is negative
- **WHEN** a deferrable load is placed
- **THEN** exporting that energy earns no carbon credit, so local consumption wins on the
  carbon objective as it already does on cost and self-consumption

#### Scenario: The rule bites only on negative prices

- **GIVEN** the same objective and an hour whose effective feed-in price is zero or above
- **WHEN** the same load is placed
- **THEN** the export credit applies unchanged, so the only behaviour this requirement
  alters is the negative-price case

> Source: **no thread source and no production system states this.** It is the owner's
> reasoning alone — a negative feed-in price is the market refusing the power — decided by
> the owner 2026-08-02 (D20, docs/OWNER_DECISIONS.md), which records it as the most
> overturnable decision in the set; the decision pack declined to recommend it for exactly
> that reason, and every other requirement in this corpus rests on a #3478 comment or a
> production system, so this one is deliberately marked as the exception. The cost side
> needed no such rule: valuing a kilowatt-hour at the consumption price imported and the
> feed-in price exported (`define-price-providers` _Feed-in pricing_) already makes cost and
> self-consumption agree when the feed-in price is negative. Alternatives preserved in
> design.md §5.

### Requirement: Objective extensibility

The objective SHALL be an extension point, so contributed algorithms and scripts can add
objectives beyond the built-in three without core changes.

#### Scenario: Contributed objective

- **GIVEN** a script contributing a custom objective (e.g. battery-wear-aware cost)
- **WHEN** the user selects it
- **THEN** planning uses it under the same engine guardrails as the built-ins

> Source: lsiepel — "or whatever user defined preferences"
> ([1481931215](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931215));
> mechanism via `extension-surface` and scripts-first-class in `engine-contract`.
