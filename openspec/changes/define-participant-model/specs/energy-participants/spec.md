# energy-participants

## ADDED Requirements

### Requirement: Declaring participants on existing Items

The system SHALL let existing Items be declared as energy participants (provider or
consumer) — by users through an `energy:` item metadata namespace and by add-ons and
scripts programmatically, both behind one provider interface — without introducing a new
hardware abstraction, so that "logical devices" wired via HTTP, rules or any binding
participate equally.

#### Scenario: Marking a wallbox consumer

- **WHEN** a user declares an existing `Number` Item that controls a wallbox as an
  energy consumer
- **THEN** the energy management engine discovers it without any binding-specific code
  and includes it in its planning

#### Scenario: No Thing required

- **WHEN** a device is modeled only as Items (no Thing)
- **THEN** it can still be declared as a participant

#### Scenario: An add-on contributes the same shape

- **GIVEN** an add-on that registers a participant programmatically instead of writing
  item metadata
- **WHEN** the engine enumerates participants
- **THEN** it sees a participant of the same shape as a user-declared one, and nothing
  above the provider interface depends on which mechanism produced it

> Source: Kai's sketch — wiring at item level, `energy:` metadata namespace, explicit
> rejection of Thing-bound profiles ([1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374),
> [1481931263](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931263));
> decided by the owner 2026-08-02 (D5, docs/OWNER_DECISIONS.md) — alternatives preserved
> in design.md §1.

### Requirement: Participant identity

Every participant SHALL be identified by the name of the Item carrying its declaration,
overridable by an explicit id in the declaration itself, so that statements made about
one device by different mechanisms are recognised as describing one participant — a
second declaration of an identity already present being a further statement about that
participant, never a second participant and never an error.

#### Scenario: Two mechanisms describe one device

- **GIVEN** a participant declared by the user on an Item and the same participant
  contributed by an add-on under that Item's name
- **WHEN** both declarations are present
- **THEN** the system sees one participant carrying two statements, so the precedence
  rule can be applied at all, instead of two unrelated participants

#### Scenario: A duplicate declaration is not an error

- **GIVEN** a participant already declared under an identity
- **WHEN** a second declaration of the same identity arrives
- **THEN** it is recorded as a further statement about that participant, and neither
  declaration is rejected or reported as a conflict

#### Scenario: Identity survives a restart

- **GIVEN** a declared participant that engine-held state refers to (a chosen priority, a
  reported declaration gap)
- **WHEN** openHAB restarts and the same declaration is read again
- **THEN** the participant presents the same identity, so that state still refers to it

> Source: surfaced by the wave-1 prototype (A12 / B8, docs/PROTOTYPE_FEEDBACK.md) — it is
> required by _Declaring participants on existing Items_ (several mechanisms describing one
> device), by `extension-surface`'s _Multiple contributors, user selection_ and by
> `participant-discovery`'s _Explicit declaration wins_, none of which can be satisfied
> unless "the same participant" is decidable; not stated in the thread; core precedent —
> `MetadataKey` is already `(namespace, itemName)`, so a contributor can address a
> user-declared participant with no coordination; decided by the owner 2026-08-02 (D5,
> docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §5.1. The precedence chain
> that resolves the statements this identity collects (explicit metadata over contributed
> over discovered, ties by `service.ranking`) and its deliberate exception for the actuation
> sink are stated once, in `extension-surface`'s _Deterministic resolution between
> contributors_ and _The actuation sink is named, never ranked_, rather than restated here.

### Requirement: Provider roles

The system SHALL model energy providers with a role of grid, PV, or battery under one
centrally fixed sign convention — grid positive = export, PV positive = producing,
battery positive = charging, consumers positive = consuming, a controllable provider's
setpoint taking the same sign as its own reading so that a positive battery setpoint
commands charging — with a device that reports the opposite sign normalised at the edge by
a declared invert flag rather than by a convention of its own.

#### Scenario: PV and grid declared

- **GIVEN** a PV production Item and a grid meter Item declared as providers
- **WHEN** the engine evaluates
- **THEN** it computes current surplus without further configuration

#### Scenario: Battery reading enters site-load math

- **GIVEN** a battery provider reporting +2 kW
- **WHEN** two calculations that both consume site load are run over that reading
- **THEN** both read it as 2 kW being stored, because the battery role's sign is fixed
  centrally rather than per declaration

#### Scenario: A setpoint follows its own reading

- **GIVEN** a controllable battery whose reading is positive while charging
- **WHEN** the engine commands +3 kW
- **THEN** the battery charges at 3 kW, the command and the reading never pointing in
  opposite directions

#### Scenario: An inverter that disagrees

- **GIVEN** a hybrid inverter whose battery channel reports charging as a negative number
- **WHEN** its provider is declared with the invert flag
- **THEN** the engine sees the corpus convention at its own boundary, and nothing above
  the declaration knows the device disagreed

> Source: lsiepel's provider module ([1481931215](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931215)),
> Kai's needs #5/#6 ([1481931209](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931209)),
> sign convention proven in emsmanager ([ENERGY_TAXONOMY](https://github.com/stamateviorel/openhab-binding-emsmanager/blob/main/docs/ENERGY_TAXONOMY.md));
> sharpened after the wave-1 prototype surfaced E6 (see docs/PROTOTYPE_FEEDBACK.md), which
> found that surplus cannot be computed at all while only the grid role's sign is fixed;
> the per-role and setpoint signs decided by the owner 2026-08-02 (D11,
> docs/OWNER_DECISIONS.md) — **and this decision inverts the battery direction the decision
> pack recommended**, so both directions and the pack's self-checking corollary are
> preserved in design.md §5.11. What is built on these signs — surplus as grid export plus
> reclaimable battery charging (D10) — is stated in `define-energy-levels` _Surplus
> escalation of the current level_ and `define-optimization-objectives` _Selectable
> objective_, with the open sub-questions in `define-engine-contract` design.md §11.

### Requirement: Controllable providers

The system SHALL support controllable providers — a battery is the primary case —
carrying a setpoint target, a mandatory `[min, max]` power clamp, a state-of-charge reading
and a priority on the same scale consumers use (lower is the better priority, defaulting to
100), so that whether its charging power is reclaimable by a given consumer is decidable
from the cycle's own inputs, and acting as a "negative load" when grid cost is high.

#### Scenario: Battery as negative load

- **GIVEN** a declared battery provider with charge above its reserve
- **WHEN** grid prices are high
- **THEN** the engine may command the battery to discharge within its declared clamp

#### Scenario: Whose charge is reclaimable

- **GIVEN** a battery charging at 3 kW with the default priority 100, a consumer at
  priority 40 and a second consumer at priority 140
- **WHEN** the engine decides whether that charging power counts as surplus for each
- **THEN** it is reclaimable for the priority-40 consumer and not for the priority-140 one,
  the comparison being against a number the battery declares rather than against an
  assumption

#### Scenario: A controllable provider without a clamp

- **GIVEN** a provider declared controllable but carrying no `[min, max]`
- **WHEN** the declaration is read
- **THEN** the participant is skipped whole and reported, rather than accepted with a
  setpoint the engine could write without bound

> Source: Kai's need #8 and controllable-provider commands
> ([1481931209](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931209),
> [1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374));
> the mandatory clamp decided by the owner 2026-08-02 (D16 / pack A10,
> docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §5.8. The provider priority
> is a **reconciliation, not a decision put to the owner**: D10 makes surplus include "the
> battery charging the engine can reclaim" and both requirements that consume that quantity
> turn on a consumer's priority being better than the battery's, while _Priority_ gives a
> priority only to **consumers** — so reclaimability had no decidable rule. It is given the
> consumer scale and the consumer default rather than a mechanism of its own, because that
> is the smallest thing that makes D10 computable; the alternative — the two-quantity
> `surplus` / `allocatableBudget` split of `define-engine-contract` design.md §11 — answers
> the same question by treating the battery as a participant the allocator serves by
> priority, and is preserved there in full.

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

### Requirement: Declared bounds in power or current

A declared bound SHALL be accepted in either dimension — power or current — on consumers
and on controllable providers alike, the engine converting a current bound to power with
the declared nominal voltage and the participant's phase count and holding watts as its
canonical internal unit, so a declaration can say what the hardware's own datasheet says.

#### Scenario: Wallbox declared in amps

- **GIVEN** a three-phase wallbox declared with min 6 A and max 32 A
- **WHEN** the engine allocates against a budget expressed in watts
- **THEN** it converts the bounds itself, without the user restating them in watts

#### Scenario: Two consumers, two dimensions, one comparison

- **GIVEN** one consumer bounded in amps and another in watts under the same budget
- **WHEN** the engine books both
- **THEN** both are compared in watts, the canonical unit, and neither declaration is
  privileged over the other

#### Scenario: Declared configuration versus item state

- **GIVEN** a bound declared as the configuration value 16 with the schema's own unit and
  a control item whose state is a quantity in milliamperes
- **WHEN** the engine compares the state against the bound
- **THEN** the comparison is made on quantities, so 16000 mA and 16 A compare equal, and
  no unit string is matched textually

> Source: surfaced by the wave-1 prototype (A9 / B4, docs/PROTOTYPE_FEEDBACK.md) — _Four
> consumer profile classes_ calls Controllable a continuous **power** profile while its own
> scenario is "min 6 A and max 32 A"; core precedent — `ConfigDescriptionParameter` carries
> `unit`/`unitLabel` for numeric parameters only, which is core stating that configuration
> is a bare number plus a schema unit while item state is a `QuantityType`; 6 A is the
> IEC 61851 minimum single-phase charging current, so restating the scenario in watts would
> make it lie about the device; decided by the owner 2026-08-02 (D16 / pack A10,
> docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §5.9.

### Requirement: Consumer power figure

The model SHALL fix the power figure that budget-constrained scheduling and the
electrical-limit floor book against a consumer by its profile class — Controllable: its
declared maximum; Batch: its rated power scaled by its curve, the curve's peak when
admitting it; Simple: a declared `ratedPower`, optional, falling back to the surplus
on-threshold when absent and reporting that fallback as a declaration gap; and
ModeControllable: none, mode changes being exempt from the planner's budget unless the
declaration carries an optional per-mode draw — rather than leaving each engine to infer
one.

#### Scenario: Two loads under one budget

- **GIVEN** two consumers whose class-determined figures together exceed the site's power
  budget
- **WHEN** the engine allocates within that budget
- **THEN** it books each consumer's figure and admits only what fits

#### Scenario: A load that has not started yet

- **GIVEN** a consumer that is off and therefore draws nothing
- **WHEN** the engine considers starting it inside a power budget
- **THEN** it books that consumer's declared figure for the allocation, no measurement of
  the load being possible before it starts

#### Scenario: Simple consumer without a declared rating

- **GIVEN** a Simple consumer declaring a surplus on-threshold and no `ratedPower`
- **WHEN** the engine books it
- **THEN** it falls back to the on-threshold and reports the participant as carrying a
  declaration gap, and the declaration is not rejected

#### Scenario: Mode change carries no figure

- **GIVEN** a ModeControllable heat pump with no per-mode draw declared
- **WHEN** an algorithm proposes a mode change
- **THEN** the planner's budget does not book a figure for it, and the consequence is
  caught by the runtime floor through measurement on a later cycle

> Source: surfaced by the wave-1 prototype (E4, docs/PROTOTYPE_FEEDBACK.md) — it is
> required by `grid-constraints`' _Load balancing under a power budget_ ("their declared
> maximum power") and by `engine-contract`'s _Electrical limits outrank optimization_,
> while the only per-consumer power value previously stated here was the Simple class's
> surplus on-threshold; not stated in the thread; production precedent — `PowerProfile.Batch`
> already carries `ratedW` separately from any switching threshold, and the mode exemption is
> a limitation the reference names honestly ("mode loads' draw is unknown to the surplus
> budget"); decided by the owner 2026-08-02 (D15, docs/OWNER_DECISIONS.md) — alternatives
> preserved in design.md §5.13.

### Requirement: Declared power measurement

A participant SHALL be able to declare an Item carrying its live power, which the engine
uses in preference to that participant's declared figure wherever a fresher measurement
answers the question being asked — the electrical-limit floor booking the larger of the two
while the participant is running and the declared figure while it is not
(`define-engine-contract` _What the limit floor books_) — and never in place of the
declared figure when admitting a load that is not yet running.

#### Scenario: A running load is booked at what it draws

- **GIVEN** a running consumer declaring a measurement Item that reads above its declared
  figure
- **WHEN** the engine books it against the electrical-limit floor
- **THEN** the measurement is used, so a device that ignores its envelope is visible to
  the floor rather than hidden by its own declaration

#### Scenario: An idle load is booked at its declaration

- **GIVEN** a consumer that is off, declaring both a figure and a measurement Item
  reading zero
- **WHEN** the engine considers admitting it
- **THEN** the declared figure is booked, because a measurement of a load that has not
  started says nothing about what it will draw

#### Scenario: No measurement declared

- **GIVEN** a consumer declaring no measurement Item
- **WHEN** the engine books it
- **THEN** it uses the declared figure and does not treat the absence as an error

> Source: surfaced by the wave-1 prototype (N1, docs/PROTOTYPE_FEEDBACK.md) — a
> per-consumer measurement item name was load-bearing in three places of the prototype with
> no requirement behind it, and the corpus's only hook, `extension-surface`'s _Graceful
> degradation on contributor loss_, is site-level and never per consumer; production
> precedent — the reference's `measure` on any consumer, "running-detection and
> delivered-energy metering trust measurement over commanded state"; decided by the owner
> 2026-08-02 (D15, docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §5.14,
> which also keeps the broader reading this wording narrows: **prefer the measurement
> unconditionally once running**, which pack A6 states in the same breath and which
> disagrees with `define-engine-contract` _What the limit floor books_ whenever a running
> load measures **below** its declared figure.

### Requirement: Phase declaration

A consumer SHALL declare the phases it draws on as the integer indices 1, 2 and 3 — a
provider likewise being able to declare per-phase readings — so per-phase electrical
limits are enforced against the loads that occupy each phase, a participant declaring
none being exempt from per-phase enforcement, still constrained by the site total, and
reported as a declaration gap wherever the site declares per-phase budgets.

#### Scenario: Single-phase load on the loaded phase

- **GIVEN** a consumer declaring that it draws on phase 1
- **WHEN** that phase is near its limit while the others are not
- **THEN** the engine constrains that consumer, instead of seeing only a site total

#### Scenario: Three-phase load

- **GIVEN** a wallbox declaring phases 1, 2 and 3
- **WHEN** the engine computes per-phase headroom
- **THEN** the wallbox's draw is attributed to every phase it declared

#### Scenario: A consumer that declares no phase

- **GIVEN** a site declaring per-phase budgets and a consumer declaring no phase
- **WHEN** the engine computes per-phase headroom
- **THEN** that consumer is not attributed to any phase — neither to all three nor to a
  guessed one — is reported as a declaration gap, and remains constrained by the site
  total

#### Scenario: Per-phase provider reading

- **GIVEN** a grid provider declaring a reading per phase
- **WHEN** the engine computes headroom for the phase nearest its limit
- **THEN** load the engine does not itself control is visible on that phase, rather than
  only what the engine dispatched

> Source: surfaced by the wave-1 prototype (E2 / E3, docs/PROTOTYPE_FEEDBACK.md) — it is
> required by `engine-contract`'s _Electrical limits outrank optimization_, whose
> "Phase-aware headroom" scenario opens with "a consumer that declares which phase(s) it
> draws on" while no requirement provided for it; not stated in the thread; production
> precedent — the reference carries `totalAmpsL1/L2/L3` and computes real per-phase
> headroom from measured per-phase load; decided by the owner 2026-08-02 (D16 / pack A10,
> docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §5.12.

### Requirement: Simple-consumer protection parameters

The Simple class SHALL support the full protection set proven in production: a
per-consumer surplus on-threshold, minimum and maximum ON runtime, minimum OFF time
(cooldown), and maximum OFF time (duty-cycle guarantee) — with the minimum runtime
doubling as the catch-up time after a forced restart, which is observed as any OFF→ON
transition the engine did not command.

#### Scenario: Fridge duty-cycle guarantee

- **GIVEN** a cooling device that declares a maximum OFF time of 45 minutes
- **WHEN** 45 minutes have passed since it went OFF
- **THEN** the engine switches it back ON, regardless of price or surplus

#### Scenario: Cooldown respected

- **WHEN** a compressor load declares a minimum OFF time
- **THEN** the engine never switches it back ON before that time has elapsed

#### Scenario: A start the engine did not command

- **GIVEN** a protected consumer that someone else switches from OFF to ON
- **WHEN** the engine observes that transition
- **THEN** the minimum runtime applies from it exactly as it would from an
  engine-initiated start, without the engine having to recognise why the device started

> Source: storm.house AN-AUS consumers + weisseWare cooling rules, carried over on
> mstormi's request ([5012316336](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5012316336)),
> ported and confirmed in [5014707374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5014707374);
> the forced restart restated observably, decided by the owner 2026-08-02 (D7 refinement,
> docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §5.15.

### Requirement: Reading staleness declaration

A participant SHALL be able to declare a maximum age for the readings it contributes,
with an unreadable state and the `UNDEF` and `NULL` states counting as stale whether or
not an age is declared, so a reading that stopped moving can be told from a fresh one
without the framework inventing an ageing mechanism of its own.

#### Scenario: Declared age exceeded

- **GIVEN** a participant declaring a maximum reading age of 30 seconds
- **WHEN** its reading has not updated for longer than that
- **THEN** the engine treats that reading as stale

#### Scenario: Undefined state with no declared age

- **GIVEN** a participant declaring no maximum age
- **WHEN** its reading becomes `UNDEF`
- **THEN** the engine treats that reading as stale anyway

#### Scenario: A frozen item made observable

- **GIVEN** a source that stops updating without ever going undefined, on a site that
  configures core's `expire` namespace on that Item
- **WHEN** the expire duration elapses
- **THEN** the Item reads `UNDEF` and the reading counts as stale through the same rule,
  with no EMS-specific ageing mechanism involved

> Source: surfaced by the wave-1 prototype (E12, docs/PROTOTYPE_FEEDBACK.md) and required
> by `extension-surface`'s _Graceful degradation on contributor loss_, whose "Safety input
> lost" scenario turns on a reading having "gone stale" while nothing said what that means;
> core precedent — `ExpireManager`'s `expire` namespace already converts "silently froze
> hours ago" into "unreadable"; production precedent — the reference's 30-second staleness
> gate on the live-measurement bridge; decided by the owner 2026-08-02 (D14,
> docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §5.17.

### Requirement: Acknowledgement declaration

A participant SHALL be able to declare an acknowledgement window and an acknowledgement
tolerance band — the band being an absolute quantity in the control Item's own dimension,
never a fraction of the commanded value — both overriding the engine's own defaults for
that participant alone.

#### Scenario: A wallbox that never lands exactly

- **GIVEN** a wallbox whose current-limit Item settles within 1 mA of any commanded value
- **WHEN** its declaration carries a tolerance band of 0.01 A
- **THEN** a reported 15.999 A counts as an acknowledgement of a commanded 16 A, on a
  comparison made in amperes rather than as a percentage of 16

#### Scenario: A slow device gets a longer window

- **GIVEN** a device that takes two minutes to reflect a command, on an engine whose
  acknowledgement window defaults to 60 seconds
- **WHEN** its declaration carries a window of three minutes
- **THEN** that window applies to this participant and the engine's default applies to
  every other

#### Scenario: Neither is required

- **GIVEN** a participant declaring no window and no tolerance band
- **WHEN** the engine tests for acknowledgement
- **THEN** the engine's own default window applies and the comparison is exact under
  units, the absence of a declaration not being an error

> Source: required by `define-engine-contract` _Acknowledgement-aware actuation_, which
> counts a reading "within a tolerance band where the participant declares one" and takes a
> window "overridable per participant" while this change — the corpus's only enumeration of
> what a participant may declare — provided for neither; production precedent — the
> reference's ACK window exists because OCPP chargers report lagging, imprecise values, and
> its 15.999 A against 16 A is the corpus's own worked example. The declarations follow from
> the owner's decisions of 2026-08-02 (D8 for the tolerance band counting at all, D17 · row
> EC-3a for the window being per-participant overridable, docs/OWNER_DECISIONS.md) —
> alternatives preserved in `define-engine-contract` design.md §3. The band's **value** is
> deliberately still open there, recorded by the owner as a parameter yet to be set; its
> **dimension** is fixed here because an absolute band and a proportional one differ by two
> orders of magnitude on this very example, and neither the decision nor the pack chose
> between them.

### Requirement: Demand declaration

The system SHALL let any consumer declare future demand in exactly three terms — an energy
amount, a deadline that resolves to exactly one instant each time the engine evaluates it,
and a flag marking the demand as one that must not be interrupted (e.g. "4 kWh ready by
07:00") — with Batch-class demand additionally carrying a load curve so scheduling can
account for when within the program the power is drawn.

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

#### Scenario: A state-of-charge target is converted before it is declared

- **GIVEN** an EV whose owner thinks in "charged to 80 % by noon" and a declaring add-on
  that knows the pack's capacity and its present charge
- **WHEN** the demand is declared
- **THEN** the engine receives an energy amount and a deadline, the conversion having
  happened where the capacity is known, and no requirement obliges the engine to know
  battery capacities

> Source: jlaur's founding use case ([issue body](https://github.com/openhab/openhab-core/issues/3478),
> [1481931141](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931141)),
> Kai's need #3 ([1481931209](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931209)),
> masipila's consecutive stories ([1481931272](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931272));
> sharpened after the wave-1 prototype surfaced A2 (see docs/PROTOTYPE_FEEDBACK.md) —
> which deadline forms are admissible (absolute, recurring, both) stays open, design.md
> §5.3, as does where the load curve belongs (§5.5). The three demand terms, and the
> removal of the two examples the shape could not express, are a stated default of the
> owner's parameter pass, decided 2026-08-02 (D17, Part B row PM-5.4,
> docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §5.4.

### Requirement: Priority

Every consumer SHALL carry a priority in which the lower number is the better priority —
served first when available power cannot serve everyone, and winning when two decisions
conflict — defaulting to 100 where none is declared, and tie-broken between equal
priorities by participant id ascending, so the order is total, deterministic, computable
from the cycle's own inputs, and never dependent on the order in which participants
happen to be enumerated.

#### Scenario: Two consumers, limited surplus

- **GIVEN** two eligible consumers, one at priority 20 and one at priority 60
- **WHEN** PV surplus suffices for only one of them
- **THEN** the priority-20 consumer is served, and the allocation order does not depend on
  incidental iteration order

#### Scenario: Equal priorities are broken by participant id

- **GIVEN** two eligible consumers both at priority 50
- **WHEN** surplus suffices for only one of them
- **THEN** the one whose participant id sorts first ascending is served, rather than the
  one that happened to be enumerated first

#### Scenario: Consumer that declares no priority

- **GIVEN** a consumer declared without an explicit priority and another at priority 50
- **WHEN** the engine orders consumers
- **THEN** the undeclared one is placed at 100, behind the priority-50 consumer, not by
  the incidental order in which it was declared

#### Scenario: The tie-break carries nothing between cycles

- **GIVEN** two equal-priority consumers, one of which was served in the previous cycle
- **WHEN** the same inputs are evaluated again
- **THEN** the same consumer is served again, the tie-break being a function of the
  cycle's inputs and not of what was served last time

> Source: Kai's need #7 ([1481931209](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931209)),
> storm.house priority parameter — adopting it fixed a real ordering bug in the
> reference ([5014707374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5014707374));
> sharpened after the wave-1 prototype surfaced A4 / E1 / B7 (see
> docs/PROTOTYPE_FEEDBACK.md); production precedent — `sortByPriority` sorts ascending,
> ties by id, `DEFAULT_PRIORITY = 100`; decided by the owner 2026-08-02 (D4,
> docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §5.2, which also records
> the known follow-on: the reference binding's live `Controller` contract states the
> opposite direction for conflicts and has to be reconciled.

### Requirement: Level-to-mode mapping for any mode count

A ModeControllable consumer with _n_ ordered modes SHALL take its mode from the site
level by the stated mapping `modeIndex = ceil(level × (n − 1) / 3)`, where `modeIndex` is
a zero-based position in that consumer's own ordered list and index 0 is its most
restricted mode — a two-mode consumer being excluded from the mapping and steered instead
by a declared "on at level ≥ N" gate of the form _Level-gated operation_ states, so that a
binary device's threshold is the one its owner declared rather than one the formula
implies.

#### Scenario: Four modes need no translation

- **GIVEN** a heat pump declaring the four ordered modes blocked / normal / encouraged /
  forced
- **WHEN** each of the four site levels drives it in turn
- **THEN** it lands in the mode at the matching position — level `0` in index 0 up to
  level `3` in index 3 — so the four-mode case the SG-ready scenario describes needs no
  translation logic in user rules

#### Scenario: Three modes, no mode skipped

- **GIVEN** a device declaring three ordered modes
- **WHEN** the level runs from blocked to overcapacity
- **THEN** blocked selects index 0, normal selects index 1 rather than being floored into
  the most restricted mode, and encouraged and overcapacity both select index 2

#### Scenario: A two-mode device is gated, not mapped

- **GIVEN** a two-mode device whose owner declared "on at encouraged or above"
- **WHEN** the level is normal
- **THEN** it stays in its restricted mode, because the mapping is not applied to a binary
  device — applying it would place such a device in its permissive mode at every level
  above blocked and silently outrank the declared gate

#### Scenario: An index is not a wire value

- **GIVEN** an SG-ready participant driven from index 3
- **WHEN** the mode is written to the device
- **THEN** SG-ready mode 4 is asserted through the named correspondence
  (`define-energy-levels` _SG-ready mode mapping_), never by arithmetic that adds one to
  an index or a level code

> Source: surfaced by the wave-1 prototype (A6 ≡ L2, docs/PROTOTYPE_FEEDBACK.md) — _Four
> consumer profile classes_ puts no bound on the number of modes while the mapping is
> specified for four only, and the prototype left it as engine policy, i.e. unimplemented
> rather than implemented differently; production precedent — `EnergyManagementService`
> computes this exact expression on a live building; decided by the owner 2026-08-02 as a
> stated default (D17, Part B row PM-5.10, docs/OWNER_DECISIONS.md) — alternatives
> preserved in design.md §5.10 and in `define-energy-levels` design.md §3. One edge this
> requirement names but does not close: _Level-gated operation_ grants the "on at level ≥ N"
> gate to **Simple** consumers only (D13 deliberately kept it there), so whether a two-mode
> ModeControllable consumer may declare that gate, or whether a two-state device is declared
> Simple in the first place, is recorded as **undecided** in design.md §5.10 — the sentence
> above tells an implementer which control steers a binary device, not which class may
> declare it.

### Requirement: Level-gated operation

A Simple consumer SHALL be able to declare the minimum site energy level at which the
engine runs it ("on at level ≥ N") as its coupling to the shared level signal, a gate the
engine itself enforces for every algorithm rather than offering as advice a contributed
one may set aside.

#### Scenario: Pool pump waits for encouraged

- **GIVEN** a pool pump declaring "run at encouraged or above"
- **WHEN** the site level rises to encouraged
- **THEN** the engine may switch it ON, and switches it OFF again when the level drops
  below (protection times still respected)

#### Scenario: A contributed algorithm cannot lower the gate

- **GIVEN** a consumer declaring "run at encouraged or above" and a contributed algorithm
  proposing to start it while the level is normal
- **WHEN** the cycle is dispatched
- **THEN** the device is not started, because the user-declared gate is engine-owned

> Source: storm.house "Schaltniveau" (AN-AUS consumers,
> [docs](https://storm.house/docs/#AN-AUS_Verbraucher),
> [5012316336](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5012316336));
> carried over and confirmed in-thread — "schaltniveau (on at energy level >= N)"
> ([5014707374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5014707374));
> the level scale itself is defined in `energy-levels`; the gate's engine-owned status and
> the removal of `never` from this requirement decided by the owner 2026-08-02 (D13,
> docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §5.7.

### Requirement: Hands-off consumers

Any consumer, of any of the four profile classes, SHALL be able to declare itself
hands-off — the `never` flag other requirements bind by that name — after which the engine
neither starts, stops, trims nor re-modes it while still reading everything it declares.

#### Scenario: Manual-only device

- **GIVEN** a consumer declaring itself hands-off
- **WHEN** any level is reached and any algorithm proposes an action for it
- **THEN** the engine does not steer it

#### Scenario: Hands-off on a class other than Simple

- **GIVEN** an EV charger declared Controllable and a dishwasher declared Batch, both
  marked hands-off
- **WHEN** the engine evaluates
- **THEN** neither is trimmed, started nor deferred, the flag applying to every profile
  class and not only to Simple

#### Scenario: A hands-off device is still measured

- **GIVEN** a hands-off consumer declaring a measurement Item
- **WHEN** the engine enforces the electrical-limit floor
- **THEN** that consumer's draw is still counted against the limits, so marking a device
  hands-off never costs the floor its visibility of the load

#### Scenario: An overload cannot steer a hands-off load

- **GIVEN** an overload whose worst-priority load is marked hands-off
- **WHEN** the floor resolves it, and likewise when a safety input goes stale
- **THEN** that load is booked as load the others are trimmed against and is neither
  trimmed, deferred nor switched off — as a running Batch programme is — and the floor
  refuses with a reason where the remainder still does not fit

> Source: storm.house scopes the gate per consumer regardless of class
> ([5012316336](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5012316336),
> [docs](https://storm.house/docs/#AN-AUS_Verbraucher)); surfaced by the wave-1 prototype
> (A1 / F13, docs/PROTOTYPE_FEEDBACK.md) as a class-scope defect — an EV or a dishwasher
> could not be marked hands-off at all; lifting it off the Simple profile onto the consumer
> decided by the owner 2026-08-02 (D13, docs/OWNER_DECISIONS.md) — alternatives preserved
> in design.md §5.7. D13 placed this prohibition against **contributed algorithms**; that
> left the engine's own electrical-limit floor and safe state dispositioning every load with
> no carve-out for it, so the scope is made definite here — the flag binds the engine too,
> on the same terms a running Batch programme gets. The reading in which **hands-off binds
> algorithms but yields to the electrical-limit rung**, so a fuse may shed a hands-off load,
> is preserved in design.md §5.7 and named in `define-engine-contract` _Constraint
> precedence ladder_.

### Requirement: Readiness interlock

A consumer SHALL be able to declare a readiness condition (a "start-ready" flag) that
gates any engine-initiated start, enforced by the engine for every algorithm, so devices
are only steered when physically prepared.

#### Scenario: Not ready, not started

- **GIVEN** a pool pump whose readiness Item is OFF
- **WHEN** surplus would otherwise start it
- **THEN** the engine does not start it, without treating this as an error

> Source: storm.house "startklar" ([5012316336](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5012316336),
> [docs](https://storm.house/docs/#AN-AUS_Verbraucher)); its engine-owned status decided by
> the owner 2026-08-02 (D13, docs/OWNER_DECISIONS.md) — alternatives preserved in
> `define-engine-contract` design.md §16.

### Requirement: Malformed declarations are reported, never partially accepted

A declaration the system cannot read completely SHALL cause the whole participant to be
skipped and reported on the framework's configuration-status surface, keyed on the Item
that carries it, rather than being partially accepted or read as some default class.

#### Scenario: One unreadable key, whole participant skipped

- **GIVEN** a consumer declaration whose protection parameters are readable but whose
  bounds are not
- **WHEN** the declaration is read
- **THEN** the participant is not registered at all and the problem is reported against
  its Item, rather than the engine steering the device on half an intent

#### Scenario: An unrecognised profile class is not defaulted

- **GIVEN** a declaration naming a profile class the framework does not know
- **WHEN** the declaration is read
- **THEN** it is reported as a configuration error and the participant is skipped, and it
  is not read as Simple

#### Scenario: The report is machine-readable

- **GIVEN** a participant skipped for a malformed declaration
- **WHEN** a UI or a rule asks the framework about that Item's configuration status
- **THEN** the problem is available there as a configuration-status message, not only as
  a line in the log

> Source: surfaced by the wave-1 prototype (B11 / B12, docs/PROTOTYPE_FEEDBACK.md) — the
> corpus names no surface on which a declaration in error can appear; core precedent —
> `ConfigStatusProvider` with `ConfigStatusMessage{INFORMATION,WARNING,ERROR}` and
> `ConfigStatusInfoEvent` already exist, are i18n'd and are consumed by UIs, so an
> EMS-specific error surface would be the reviewable defect; decided by the owner
> 2026-08-02 (D16 / pack A8, docs/OWNER_DECISIONS.md) — alternatives preserved in
> design.md §5.18.
