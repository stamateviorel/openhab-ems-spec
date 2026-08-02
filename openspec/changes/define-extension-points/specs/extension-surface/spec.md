# extension-surface

## ADDED Requirements

### Requirement: Runtime contribution

The system SHALL let separately installed add-ons and user scripts contribute energy
participants, data providers, profiles and algorithms at runtime — registered when the
contributor appears and unregistered when it goes away — without modifying or restarting
the core.

#### Scenario: Price binding installed

- **GIVEN** a running EMS with no price source
- **WHEN** the user installs a price-provider add-on
- **THEN** its provider becomes available to the engine without a core change or restart

#### Scenario: Script-contributed consumer

- **GIVEN** a user script that computes an individual demand description
- **WHEN** the script registers it
- **THEN** the engine treats it exactly like an add-on-contributed one

> Source: Kai — "diverse implementations and sources … ideally implemented by bindings,
> but we could also have implementations that are done through scripts"; consumers
> "likely to be done through fairly individual scripts"
> ([1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374));
> "many bindings that each bring their own `EnergyProvider`"
> ([1483930195](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1483930195)).

### Requirement: Expressive declaration surface

Whatever mechanism carries a participant declaration SHALL be able to express every
participant attribute these requirements define, each declared physical quantity being
unambiguous as to its unit.

#### Scenario: An attribute a requirement defines can be declared

- **GIVEN** requirements that define a Controllable consumer's minimum and maximum, a
  demand whose hours must not be interrupted, and a minimum site level at which a
  consumer may run
- **WHEN** a site declares such a consumer through a supported mechanism
- **THEN** each of those attributes can be stated in the declaration itself, rather than
  inferred or supplied out of band

#### Scenario: A declared quantity carries its dimension

- **GIVEN** a declaration carrying a numeric bound
- **WHEN** a contributor writes it and the engine reads it
- **THEN** both read the same unit, fixed by a stated rule — carried in the declaration or
  mandated by the specification — rather than assumed independently by each side

> Source: surfaced by the wave-1 prototype (B2, B4 — see docs/PROTOTYPE_FEEDBACK.md) — it
> is required by `energy-participants` _Four consumer profile classes_ ("Wallbox as
> Controllable with min 6 A and max 32 A"), _Demand declaration_ ("Consecutive-hours
> demand") and _Level-gated operation_, none of which any declaration surface the corpus
> defines can express; not stated in the thread.

### Requirement: Multiple contributors, user selection

When multiple contributors provide the same kind of data, the user SHALL select or
combine which contributions are used in the calculations.

#### Scenario: Two price sources, one composition

- **GIVEN** one binding providing spot prices and another providing grid tariffs
- **WHEN** the user configures the composition
- **THEN** the engine uses exactly the selected items, and an unselected source changes
  nothing

> Source: jlaur — "The user should be able to configure which items to include in the
> calculations" ([1489220814](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1489220814));
> Kai — "Yes, agreed" ([1500300143](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1500300143));
> the price-specific instance lives in `price-data` (component composition) — this
> generalizes it to every contribution kind.

### Requirement: Deterministic resolution between contributors

When two contributors declare the same participant, the collision SHALL be resolved by a
stated rule whose outcome does not depend on the order in which the contributors
registered.

#### Scenario: An add-on and a script declare the same participant

- **GIVEN** an add-on and a user script that each declare the same participant
- **WHEN** both are registered
- **THEN** the outcome is the one the stated rule prescribes, and a restart that registers
  the two in the opposite order produces that same outcome

#### Scenario: A contributor leaves and comes back

- **GIVEN** a participant declared by two contributors and already resolved
- **WHEN** one contributor is removed and later returns
- **THEN** the outcome is the one it was before that contributor left

> Source: surfaced by the wave-1 prototype (B9 — see docs/PROTOTYPE_FEEDBACK.md) — it is
> required by _Runtime contribution_, which admits add-on and script contributors on equal
> terms without saying what happens when two of them describe the same device, and by
> _Multiple contributors, user selection_, which says what may coexist but not what the
> engine does when two contributions collide; the corpus's only precedence statement,
> `participant-discovery` _Explicit declaration wins_, settles an explicit declaration
> against a **discovered** proposal in wave 4 and says nothing about a **contributed** one;
> not stated in the thread.

### Requirement: Contributor-owned complexity

Provider-specific logic — fetching, pricing math, device quirks, forecast modelling —
SHALL remain inside the contributor, with the engine-facing interface carrying only the
generic data: declarations and series of timestamped values.

#### Scenario: New provider, untouched engine

- **GIVEN** a new energy provider with an exotic upstream API
- **WHEN** it is contributed
- **THEN** the engine consumes its generic series without any provider-specific code in
  the core

> Source: lsiepel — "keep all those details under the responsibility of this
> `EnergyProvider` … keep the interface … as simple as possible. Possibly not much more
> than a list of some generic propertys and a list of datapoints holding timestamp,
> available power, cost of power … The scheduling engine can be very lean"
> ([1481931283](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931283)).

### Requirement: Graceful degradation on contributor loss

When a contributor fails or is removed, the engine SHALL continue operating with the
remaining participants and any persisted baselines rather than halting or discarding
state.

#### Scenario: Forecast source removed mid-day

- **GIVEN** planning that uses a contributed solar forecast with a persisted baseline
- **WHEN** that add-on is uninstalled or its service goes dark
- **THEN** the engine keeps planning on the baseline and the remaining participants, and
  reports the degraded source

#### Scenario: Safety input lost — degrade safe, not blind

- **GIVEN** a live power measurement that feeds the electrical-limit floor
- **WHEN** its source disappears or goes stale
- **THEN** the engine does NOT keep optimizing on stale safety data — it falls back to a
  conservative safe state for engine-steered loads (floor levels or pause) until the
  measurement returns

> Source: the baseline-fallback behaviour from mstormi — the EMS "to run well even if
> your internet connection or forecast provider fails for whatever reason, for whatever
> outage duration" ([5020830338](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5020830338)),
> extended here from data sources to any contributor; extension marked as design
> generalization. The safety-input distinction is **reference-production-sourced**
> (a staleness gate capping all loads when the live-measurement bridge goes silent) —
> data-plane loss degrades to baselines, safety-input loss degrades to safe.

### Requirement: Core-shipped defaults without privilege

The framework SHALL allow generic default implementations to ship with the core itself —
the configurable grid-price provider being the named case — usable with zero add-ons
installed and replaceable by contributed alternatives on equal terms.

#### Scenario: Works out of the box, replaceable later

- **GIVEN** a fresh installation with no energy add-ons
- **WHEN** the user configures the core's generic grid-price provider
- **THEN** the EMS is functional, and later installing a dedicated price binding can take
  over that role without migration pain

> Source: Kai — the generic `GridEnergyProvider`: "if we manage to implement this class
> in a configurable and generic way, we could possibly add this as a default
> implementation to the core directly"
> ([1483930195](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1483930195));
> the price-specific requirement lives in `price-data` — this states the architectural
> allowance.
