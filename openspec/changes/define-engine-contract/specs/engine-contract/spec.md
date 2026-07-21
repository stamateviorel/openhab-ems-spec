# engine-contract

## ADDED Requirements

### Requirement: Central periodic evaluation
The system SHALL run one central engine that periodically evaluates all declared
participants against the same consistent context snapshot and produces the cycle's
control decisions.

#### Scenario: One snapshot per cycle
- **GIVEN** two consumers eligible for surplus in the same cycle
- **WHEN** the engine evaluates them
- **THEN** both are judged against the same surplus figure, not values read at different
  moments

#### Scenario: Provider data refreshed regularly
- **GIVEN** price providers publishing future prices
- **WHEN** the engine runs
- **THEN** it re-reads provider data on a regular cadence rather than requiring push
  notifications from providers

> Source: Kai's need #4 — "a central controller that mixes this all together and
> activates/controls the different devices"
> ([1481931209](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931209));
> lsiepel's Engine module ([1481931215](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931215));
> pull cadence — "the EMS regularly scrapes the prices from the providers — possibly up
> to every minute", push deferred
> ([1482078918](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1482078918)).

### Requirement: Deterministic conflict resolution
When multiple decisions target the same device in one cycle, the engine SHALL resolve the
conflict deterministically by declared priority, independent of registration or iteration
order.

#### Scenario: Higher priority wins
- **GIVEN** a shedding decision and an optimization decision addressing the same load
- **WHEN** the cycle is dispatched
- **THEN** the higher-priority decision is applied and the outcome is identical on every
  run with the same inputs

> Source: Kai's need #7 — "load selection must prioritize and weigh the different loads
> against each other" ([1481931209](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931209));
> deterministic ordering proven in the reference (priority contract: "lower runs first,
> higher wins on conflict").

### Requirement: Electrical limits outrank optimization
The engine SHALL enforce declared electrical limits — a total power budget and, where
declared, per-phase headroom — before applying any optimization decision, so no plan can
schedule the site past its physical limits.

#### Scenario: Budget respected at dispatch time
- **GIVEN** loads whose combined draw would exceed the declared budget this cycle
- **WHEN** the engine dispatches
- **THEN** lower-priority loads are trimmed or deferred until the total fits, regardless
  of what the plan proposed

#### Scenario: Phase-aware headroom
- **GIVEN** a consumer that declares which phase(s) it draws on
- **WHEN** one phase is near its limit
- **THEN** the engine accounts for that phase specifically, not only the site total

> Source: seime — "limit the total load within the hour (or 15 minute buckets)"
> ([1481931351](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931351));
> masipila — per-phase load balancing, "you can't toggle all the devices on at the same
> time" ([1481931297](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931297));
> relation: `grid-constraints` covers cost-optimal *planning* under budgets — this is the
> runtime *floor* beneath any plan. The "regardless of the active algorithm"
> generalization is a design synthesis from the reference's safety chain, **flagged for
> review**.

### Requirement: Shadow mode
The engine SHALL provide an observe-only shadow mode — computing and logging every
decision without writing to any item — and default to it on first install.

#### Scenario: Fresh install writes nothing
- **GIVEN** a newly configured installation
- **WHEN** the engine runs its first cycles
- **THEN** decisions appear in the logs and no device is actuated until the user leaves
  shadow mode

#### Scenario: Side-by-side validation
- **GIVEN** an existing user automation still in control
- **WHEN** the engine runs in shadow next to it
- **THEN** its would-have-done decisions can be compared against the existing behaviour
  before any cutover

> Source: shadow-first validation as practiced and reported in-thread — unit tests, then
> "weeks of shadow mode next to my old production rules comparing every decision, and
> only then live control"
> ([5014707411](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5014707411));
> reference default: shadow TRUE on first install.

### Requirement: Master stop
A single master control SHALL stop all engine actuation immediately when engaged, without
requiring uninstallation or reconfiguration of participants.

#### Scenario: Stop mid-plan
- **GIVEN** the engine actively steering devices
- **WHEN** the master control is engaged
- **THEN** no further writes occur from the next cycle on, and disengaging it resumes
  normal operation

> Source: **reference production experience** (master shadow flag as the documented kill
> switch, default-safe) — not stated in the thread; flagged as such.

### Requirement: Acknowledgement-aware actuation
The engine SHALL avoid re-sending a command to a device while the previous command is
still unacknowledged, so devices whose reported state lags the command are not flooded
with repeats.

#### Scenario: Charger with lagging state
- **GIVEN** an EV charger whose current-limit item does not reflect a command until the
  device acknowledges it
- **WHEN** the engine has sent a new limit and the state still shows the old value
- **THEN** the command is not re-sent every cycle while the acknowledgement window is
  open

> Source: **reference production experience** — an ACK window preventing command
> re-sends, essential for `autoupdate="false"` OCPP devices; not stated in the thread;
> flagged as such.

### Requirement: Replaceable algorithm, scripts first-class
The optimization algorithm SHALL be replaceable by the user — implementable as scripts or
rule templates, not only as compiled code — while evaluation, limits, shadow, stop and
actuation remain engine-owned for every algorithm.

#### Scenario: Script algorithm under the same guardrails
- **GIVEN** a user replaces the default algorithm with their own script
- **WHEN** the script proposes decisions
- **THEN** they flow through the same limit enforcement, shadow gating and actuation
  semantics as the default's

> Source: Kai — the EMS "could possibly then even be as simple as a rule template, so
> that people could e.g. implement easily alternative energy management algorithms and
> also do this through scripts"
> ([1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374));
> the guardrail tail follows from the engine-owned requirements above.
