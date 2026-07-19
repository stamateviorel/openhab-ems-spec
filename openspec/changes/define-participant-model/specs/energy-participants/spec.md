# energy-participants

## ADDED Requirements

### Requirement: Declaring participants on existing Items
The system SHALL let users declare existing Items as energy participants (provider or
consumer) without introducing a new hardware abstraction, so that "logical devices"
wired via HTTP, rules or any binding participate equally.

#### Scenario: Marking a wallbox consumer
- **WHEN** a user declares an existing `Number` Item that controls a wallbox as an
  energy consumer
- **THEN** the energy management engine discovers it without any binding-specific code
  and includes it in its planning

#### Scenario: No Thing required
- **WHEN** a device is modeled only as Items (no Thing)
- **THEN** it can still be declared as a participant

> Source: Kai's sketch — wiring at item level, `energy:` metadata namespace, explicit
> rejection of Thing-bound profiles ([1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374),
> [1481931263](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931263)).

### Requirement: Provider roles
The system SHALL model energy providers with a role of grid, PV, or battery, where the
grid role carries the canonical sign convention (positive = export, negative = import)
from which surplus is derived.

#### Scenario: PV and grid declared
- **WHEN** a PV production Item and a grid meter Item are declared as providers
- **THEN** the engine can compute current surplus without further configuration

> Source: lsiepel's provider module ([1481931215](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931215)),
> Kai's needs #5/#6 ([1481931209](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931209)),
> sign convention proven in emsmanager ([ENERGY_TAXONOMY](https://github.com/stamateviorel/openhab-binding-emsmanager/blob/main/docs/ENERGY_TAXONOMY.md)).

### Requirement: Controllable providers
The system SHALL support controllable providers — a battery is the primary case —
carrying a setpoint target, a [min, max] power clamp, and a state-of-charge reading,
and acting as a "negative load" when grid cost is high.

#### Scenario: Battery as negative load
- **WHEN** grid prices are high and the battery has charge above its reserve
- **THEN** the engine may command the battery to discharge within its declared clamp

> Source: Kai's need #8 and controllable-provider commands
> ([1481931209](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931209),
> [1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374)).

### Requirement: Four consumer profile classes
The system SHALL express every consumer through one of four profile classes, chosen to
cover the device population observed across three production systems: **Simple**
(ON/OFF), **Controllable** (continuous power/current), **ModeControllable** (small
ordered set of discrete modes, no power setpoint), and **Batch** (a fixed program that
must run to completion once started).

#### Scenario: Boiler as Simple
- **WHEN** a DHW boiler switch is declared with the Simple class
- **THEN** the engine steers it strictly via ON/OFF

#### Scenario: Wallbox as Controllable
- **WHEN** a wallbox is declared Controllable with min 6 A and max 32 A
- **THEN** the engine regulates its current within that range and never below the
  minimum while charging

#### Scenario: SG-ready heat pump as ModeControllable
- **WHEN** a heat pump is declared with the ordered modes blocked/normal/encouraged/forced
- **THEN** the engine selects exactly one mode and never sends a power value

#### Scenario: Dishwasher as Batch
- **WHEN** a dishwasher program is started by the engine
- **THEN** the engine never interrupts it before the program completes

> Source: Kai's Simple/Controllable power profiles ([1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374)),
> "not so well for dishwashers … more fine-grained" ([1481931227](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931227)),
> masipila's user stories a–d ([1481931272](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931272)),
> mstormi's generic consumers ([5012316336](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5012316336)),
> the four classes as implemented and field-tested in
> [emsmanager](https://github.com/stamateviorel/openhab-binding-emsmanager/blob/main/docs/ENERGY_TAXONOMY.md).

### Requirement: Simple-consumer protection parameters
The Simple class SHALL support the full protection set proven in production: a
per-consumer surplus on-threshold, minimum and maximum ON runtime, minimum OFF time
(cooldown), and maximum OFF time (duty-cycle guarantee) — and after a forced restart the
minimum runtime SHALL act as the catch-up time.

#### Scenario: Fridge duty-cycle guarantee
- **WHEN** a cooling device declares a maximum OFF time of 45 minutes
- **THEN** the engine switches it back ON no later than 45 minutes after it went OFF,
  regardless of price or surplus

#### Scenario: Cooldown respected
- **WHEN** a compressor load declares a minimum OFF time
- **THEN** the engine never switches it back ON before that time has elapsed

> Source: storm.house AN-AUS consumers + weisseWare cooling rules, carried over on
> mstormi's request ([5012316336](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5012316336)),
> ported and confirmed in [5014707374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5014707374).

### Requirement: Demand declaration
The system SHALL let any consumer declare future demand as an energy amount with a
deadline (e.g. "4 kWh ready by 07:00", "charged to 80% by noon", "run 5 h within the
next 12 h"), and Batch-class demand SHALL additionally support a load curve so
scheduling can account for when within the program the power is drawn.

#### Scenario: Dishwasher ready by 7am
- **WHEN** a dishwasher declares its most-used program's load curve and a 07:00 deadline
- **THEN** the engine picks the start time minimizing cost over the actual curve, not
  over an assumed flat draw

#### Scenario: Consecutive-hours demand
- **WHEN** a boiler declares demand that must not be interrupted
- **THEN** the scheduled hours are consecutive

> Source: jlaur's founding use case ([issue body](https://github.com/openhab/openhab-core/issues/3478),
> [1481931141](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931141)),
> Kai's need #3 ([1481931209](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931209)),
> masipila's consecutive stories ([1481931272](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931272)).

### Requirement: Priority
Every consumer SHALL carry a priority that deterministically orders surplus allocation
and load selection when available (cheap) power cannot serve all consumers.

#### Scenario: Two consumers, limited surplus
- **WHEN** PV surplus suffices for only one of two eligible consumers
- **THEN** the one with the better priority is served, and the allocation order does not
  depend on incidental iteration order

> Source: Kai's need #7 ([1481931209](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931209)),
> storm.house priority parameter — adopting it fixed a real ordering bug in the
> reference ([5014707374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5014707374)).

### Requirement: Readiness interlock
A consumer SHALL be able to declare a readiness condition (a "start-ready" flag) that
gates any engine-initiated start, so devices are only steered when physically prepared.

#### Scenario: Not ready, not started
- **WHEN** a pool pump's readiness Item is OFF
- **THEN** the engine does not start it even in surplus, without treating this as an error

> Source: storm.house "startklar" ([5012316336](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5012316336),
> [docs](https://storm.house/docs/#AN-AUS_Verbraucher)).
