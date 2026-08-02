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

### Requirement: Participant identity

Every participant SHALL carry an identity that is stable over time and comparable across
declaration mechanisms, so statements made about one device by different mechanisms are
recognised as describing one participant.

#### Scenario: Two mechanisms describe one device

- **GIVEN** a participant declared by the user and the same participant contributed by an
  add-on
- **WHEN** both declarations are present
- **THEN** the system sees one participant carrying two statements, so a precedence rule
  can be applied at all, instead of two unrelated participants

#### Scenario: Identity survives a restart

- **GIVEN** a declared participant that engine-held state refers to (protection timers, a
  chosen priority)
- **WHEN** openHAB restarts and the same declaration is read again
- **THEN** the participant presents the same identity, so that state still refers to it

> Source: surfaced by the wave-1 prototype (A12 / B8, docs/PROTOTYPE_FEEDBACK.md) — it is
> required by _Declaring participants on existing Items_ (several mechanisms describing one
> device), by `extension-surface`'s _Multiple contributors, user selection_ and by
> `participant-discovery`'s _Explicit declaration wins_, none of which can be satisfied
> unless "the same participant" is decidable; not stated in the thread. How the identity is
> formed and what a duplicate means stay open, design.md §5.1.

### Requirement: Provider roles

The system SHALL model energy providers with a role of grid, PV, or battery, each role
carrying a stated sign convention for its power reading and — where the role is
controllable — for its setpoint, of which the grid role's is the canonical one
(positive = export, negative = import) from which surplus is derived.

#### Scenario: PV and grid declared

- **GIVEN** a PV production Item and a grid meter Item declared as providers
- **WHEN** the engine evaluates
- **THEN** it computes current surplus without further configuration

#### Scenario: Battery reading enters site-load math

- **GIVEN** a battery provider reporting a non-zero power value
- **WHEN** two calculations that both consume site load are run over that reading
- **THEN** the reading contributes the same signed quantity to each of them, by the
  convention stated for the battery role

> Source: lsiepel's provider module ([1481931215](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931215)),
> Kai's needs #5/#6 ([1481931209](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931209)),
> sign convention proven in emsmanager ([ENERGY_TAXONOMY](https://github.com/stamateviorel/openhab-binding-emsmanager/blob/main/docs/ENERGY_TAXONOMY.md));
> sharpened after the wave-1 prototype surfaced E6 (see docs/PROTOTYPE_FEEDBACK.md) —
> the per-role and setpoint signs themselves are open, design.md §5.11.

### Requirement: Controllable providers

The system SHALL support controllable providers — a battery is the primary case —
carrying a setpoint target, a [min, max] power clamp, and a state-of-charge reading,
and acting as a "negative load" when grid cost is high.

#### Scenario: Battery as negative load

- **GIVEN** a declared battery provider with charge above its reserve
- **WHEN** grid prices are high
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

- **GIVEN** a wallbox declared Controllable with min 6 A and max 32 A
- **WHEN** the engine regulates its current
- **THEN** it stays within that range and never below the minimum while charging

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

### Requirement: Consumer power figure

The model SHALL state which declared value budget-constrained scheduling and the
electrical-limit floor book against a consumer, rather than leaving each engine to infer
one.

#### Scenario: Two loads under one budget

- **GIVEN** two consumers for which the model names a value to book, whose named values
  together exceed the site's power budget
- **WHEN** the engine allocates within that budget
- **THEN** it books each consumer's named value and admits only what fits

#### Scenario: A load that has not started yet

- **GIVEN** a consumer that is off and therefore draws nothing, for which the model names
  a value to book
- **WHEN** the engine considers starting it inside a power budget
- **THEN** it books that named value for the allocation, no measurement of the load being
  possible before it starts

> Source: surfaced by the wave-1 prototype (E4, docs/PROTOTYPE_FEEDBACK.md) — it is
> required by `grid-constraints`' _Load balancing under a power budget_ ("their declared
> maximum power") and by `engine-contract`'s _Electrical limits outrank optimization_,
> while the only per-consumer power value stated here is the Simple class's surplus
> on-threshold; not stated in the thread. Which value plays this role — the on-threshold,
> a separate rating, a per-class answer, or none at all for a class the model instead
> hands to a stated policy — stays open, design.md §5.13.

### Requirement: Phase declaration

A consumer SHALL be able to declare which phase or phases of the supply it draws on, so
per-phase electrical limits are enforced against the loads that occupy each phase.

#### Scenario: Single-phase load on the loaded phase

- **GIVEN** a consumer declaring that it draws on one phase
- **WHEN** that phase is near its limit while the others are not
- **THEN** the engine constrains that consumer, instead of seeing only a site total

#### Scenario: Three-phase load

- **GIVEN** a wallbox declaring that it draws on all three phases
- **WHEN** the engine computes per-phase headroom
- **THEN** the wallbox's draw is attributed to every phase it declared

> Source: surfaced by the wave-1 prototype (E2, docs/PROTOTYPE_FEEDBACK.md) — it is
> required by `engine-contract`'s _Electrical limits outrank optimization_, whose
> "Phase-aware headroom" scenario opens with "a consumer that declares which phase(s) it
> draws on" while no requirement provided for it; not stated in the thread. How phases are
> named, what an omitted declaration means, and whether providers gain a per-phase shape
> stay open, design.md §5.12.

### Requirement: Simple-consumer protection parameters

The Simple class SHALL support the full protection set proven in production: a
per-consumer surplus on-threshold, minimum and maximum ON runtime, minimum OFF time
(cooldown), and maximum OFF time (duty-cycle guarantee) — with the minimum runtime
doubling as the catch-up time after a forced restart.

#### Scenario: Fridge duty-cycle guarantee

- **GIVEN** a cooling device that declares a maximum OFF time of 45 minutes
- **WHEN** 45 minutes have passed since it went OFF
- **THEN** the engine switches it back ON, regardless of price or surplus

#### Scenario: Cooldown respected

- **WHEN** a compressor load declares a minimum OFF time
- **THEN** the engine never switches it back ON before that time has elapsed

> Source: storm.house AN-AUS consumers + weisseWare cooling rules, carried over on
> mstormi's request ([5012316336](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5012316336)),
> ported and confirmed in [5014707374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5014707374).

### Requirement: Demand declaration

The system SHALL let any consumer declare future demand as an energy amount with a
deadline that resolves to exactly one instant each time the engine evaluates it (e.g.
"4 kWh ready by 07:00", "charged to 80% by noon", "run 5 h within the next 12 h"), with
Batch-class demand additionally carrying a load curve so scheduling can account for when
within the program the power is drawn.

#### Scenario: One declaration, one deadline instant

- **GIVEN** a consumer that declares a demand deadline
- **WHEN** the engine evaluates at a given moment
- **THEN** that declaration yields exactly one deadline instant for the evaluation, so
  scheduling never has to choose between two readings of the same declaration

#### Scenario: Dishwasher ready by 7am

- **GIVEN** a dishwasher that declares its most-used program's load curve and a 07:00
  deadline
- **WHEN** the engine schedules it
- **THEN** the start time minimizes cost over the actual curve, not over an assumed flat
  draw

#### Scenario: Consecutive-hours demand

- **WHEN** a boiler declares demand that must not be interrupted
- **THEN** the scheduled hours are consecutive

> Source: jlaur's founding use case ([issue body](https://github.com/openhab/openhab-core/issues/3478),
> [1481931141](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931141)),
> Kai's need #3 ([1481931209](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931209)),
> masipila's consecutive stories ([1481931272](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931272));
> sharpened after the wave-1 prototype surfaced A2 (see docs/PROTOTYPE_FEEDBACK.md) —
> which deadline forms are admissible (absolute, recurring, both) stays open, design.md
> §5.3, as do the demand kinds the examples imply (§5.4) and where the load curve belongs
> (§5.5).

### Requirement: Priority

Every consumer SHALL carry a priority that imposes a total and deterministic order on
surplus allocation and load selection when available (cheap) power cannot serve all
consumers — an order whose direction, whose value for a consumer that declares none, and
whose tie-break between equal priorities are each stated, and which never depends on the
order in which participants happen to be enumerated.

#### Scenario: Two consumers, limited surplus

- **GIVEN** two eligible consumers with different priorities
- **WHEN** PV surplus suffices for only one of them
- **THEN** the one with the better priority is served, and the allocation order does not
  depend on incidental iteration order

#### Scenario: Equal priorities are broken by the stated rule

- **GIVEN** two eligible consumers carrying the same priority
- **WHEN** surplus suffices for only one of them
- **THEN** the stated tie-break decides which one is served, rather than the order in
  which the two happened to be enumerated

#### Scenario: Consumer that declares no priority

- **GIVEN** a consumer declared without an explicit priority
- **WHEN** the engine orders consumers
- **THEN** it is placed by the stated value for an undeclared priority, not by the
  incidental order in which it was declared

> Source: Kai's need #7 ([1481931209](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931209)),
> storm.house priority parameter — adopting it fixed a real ordering bug in the
> reference ([5014707374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5014707374));
> sharpened after the wave-1 prototype surfaced A4 / E1 / B7 (see
> docs/PROTOTYPE_FEEDBACK.md) — the direction, the default and the tie-break themselves
> stay open, design.md §5.2.

### Requirement: Level-gated operation

A Simple consumer SHALL be able to declare the minimum site energy level at which the
engine runs it ("on at level ≥ N"), including a "never" setting for devices the engine
must leave alone, as its coupling to the shared level signal.

#### Scenario: Pool pump waits for encouraged

- **GIVEN** a pool pump declaring "run at encouraged or above"
- **WHEN** the site level rises to encouraged
- **THEN** the engine may switch it ON, and switches it OFF again when the level drops
  below (protection times still respected)

#### Scenario: Manual-only device

- **GIVEN** a consumer declaring the "never" gate
- **WHEN** any level is reached
- **THEN** the engine does not steer it

> Source: storm.house "Schaltniveau" (AN-AUS consumers,
> [docs](https://storm.house/docs/#AN-AUS_Verbraucher),
> [5012316336](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5012316336));
> carried over and confirmed in-thread — "schaltniveau (on at energy level >= N)"
> ([5014707374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5014707374));
> the level scale itself is defined in `energy-levels`.

### Requirement: Readiness interlock

A consumer SHALL be able to declare a readiness condition (a "start-ready" flag) that
gates any engine-initiated start, so devices are only steered when physically prepared.

#### Scenario: Not ready, not started

- **GIVEN** a pool pump whose readiness Item is OFF
- **WHEN** surplus would otherwise start it
- **THEN** the engine does not start it, without treating this as an error

> Source: storm.house "startklar" ([5012316336](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5012316336),
> [docs](https://storm.house/docs/#AN-AUS_Verbraucher)).
