# extension-surface

## ADDED Requirements

### Requirement: Runtime contribution

The system SHALL let separately installed add-ons and user scripts contribute energy
participants, data providers, profiles and algorithms at runtime through one provider SPI
that carries both `energy:` item metadata and programmatic registration — registered when
the contributor appears and unregistered when it goes away — without modifying or
restarting the core.

#### Scenario: Price binding installed

- **GIVEN** a running EMS with no price source
- **WHEN** the user installs a price-provider add-on
- **THEN** its provider becomes available to the engine without a core change or restart

#### Scenario: Script-contributed consumer

- **GIVEN** a user script that computes an individual demand description
- **WHEN** the script registers it
- **THEN** the engine treats it exactly like an add-on-contributed one

#### Scenario: A script reloads

- **GIVEN** a script that contributed a participant under its own `contributorId`
- **WHEN** the script is unloaded and its next load registers the same participant again
- **THEN** what the previous load added is withdrawn on unload, and the new registration
  is a further statement about the same participant rather than a second one

> Source: Kai — "diverse implementations and sources … ideally implemented by bindings,
> but we could also have implementations that are done through scripts"; consumers
> "likely to be done through fairly individual scripts"
> ([1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374));
> "many bindings that each bring their own `EnergyProvider`"
> ([1483930195](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1483930195));
> decided by the owner 2026-08-02 (D5, docs/OWNER_DECISIONS.md) — one SPI over both
> mechanisms, `contributorId`-keyed re-registration; alternatives preserved in
> design.md §1. The reload scenario rests on core's script `providersupport` unload hook, which the
> corpus previously and wrongly said did not exist (design.md §5.9).

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
- **THEN** both read the same unit, because a declared value is a bare number whose
  dimension and unit come from the published `energy:` config description — watts being
  the canonical internal unit, and a current declaration converted with the declared
  nominal voltage and phase count — rather than being assumed independently by each side

> Source: surfaced by the wave-1 prototype (B2, B4 — see docs/PROTOTYPE_FEEDBACK.md) — it
> is required by `energy-participants` _Four consumer profile classes_ ("Wallbox as
> Controllable with min 6 A and max 32 A"), _Demand declaration_ ("Consecutive-hours
> demand") and _Level-gated operation_, none of which any declaration surface the corpus
> defines can express; not stated in the thread. Unit rule decided by the owner 2026-08-02
> (D16 pack A10, with D18 Part C EP-5.1, docs/OWNER_DECISIONS.md) — alternatives preserved
> in design.md §5.2; the normative units requirement lives in `define-participant-model`.

### Requirement: Multiple contributors, user selection

When multiple contributors provide the same kind of data, the user SHALL select or
combine which contributions are used in the calculations, per role, with the highest
`service.ranking` used where the site names no preference.

#### Scenario: Two price sources, one composition

- **GIVEN** one binding providing spot prices and another providing grid tariffs
- **WHEN** the user configures the composition
- **THEN** the engine uses exactly the selected items, and an unselected source changes
  nothing

#### Scenario: Nothing configured, ranking decides

- **GIVEN** two forecast contributors registered at different `service.ranking` values and
  no per-role configuration naming either
- **WHEN** the engine reads a forecast
- **THEN** it uses the higher-ranked contributor, and naming the other one in the role's
  configuration overrides that with no code change

> Source: jlaur — "The user should be able to configure which items to include in the
> calculations" ([1489220814](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1489220814));
> Kai — "Yes, agreed" ([1500300143](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1500300143));
> the price-specific instance lives in `price-data` (component composition) — this
> generalizes it to every contribution kind; per-role configuration defaulting to
> `service.ranking` decided by the owner 2026-08-02 (D17, Part B EP-4(i),
> docs/OWNER_DECISIONS.md) on core's `StateDescriptionServiceImpl` precedent —
> alternatives preserved in design.md §4.

### Requirement: The actuation sink is named, never ranked

The component that turns engine decisions into device writes SHALL be chosen by explicit
configuration naming it — one site-wide sink, with a per-participant override — and never
by service ranking, registration order or any other race between installed components.

#### Scenario: Two adapters installed, one named

- **GIVEN** two actuation adapters installed and one of them named in the site's
  configuration
- **WHEN** the engine dispatches a decision
- **THEN** the named adapter writes it, whatever ranking the other one registered at, and
  restarting in the opposite registration order changes nothing

#### Scenario: A participant overrides the site sink

- **GIVEN** a participant whose declaration names a different adapter
- **WHEN** the engine dispatches a decision for that participant
- **THEN** the participant's adapter writes it and every other participant keeps the
  site-wide one

#### Scenario: No sink named

- **GIVEN** a site with adapters installed but none named in configuration
- **WHEN** the engine reaches dispatch
- **THEN** nothing is written and the missing selection is reported as a configuration gap
  rather than resolved by picking one

> Source: surfaced by the wave-1 prototype (N9, B10 — see docs/PROTOTYPE_FEEDBACK.md) as
> the one selection where getting it wrong moves hardware; decided by the owner 2026-08-02
> (D5's deliberate exception, docs/OWNER_DECISIONS.md) — alternatives preserved in
> design.md §5.11 (ranked list with fallback; per-participant-only selection). The
> no-sink-named scenario is the direct consequence of refusing a ranking, stated here so
> the refusal is testable; not stated in the thread.

### Requirement: Deterministic resolution between contributors

When two contributors describe the same participant — identified by the Item name it is
declared on, or by an explicit `id` overriding it — the collision SHALL resolve on one
precedence chain, explicit `energy:` metadata over contributed over discovered, with ties
between contributed declarations broken by `service.ranking`, independent of the order in
which the contributors registered.

#### Scenario: An add-on and a script declare the same participant

- **GIVEN** an add-on and a user script that each declare the same participant
- **WHEN** both are registered
- **THEN** the outcome is the one the stated rule prescribes, and a restart that registers
  the two in the opposite order produces that same outcome

#### Scenario: A contributor leaves and comes back

- **GIVEN** a participant declared by two contributors and already resolved
- **WHEN** one contributor is removed and later returns
- **THEN** the outcome is the one it was before that contributor left

#### Scenario: A second declaration is a further statement, not a second participant

- **GIVEN** a participant an add-on already contributed for an Item
- **WHEN** the user adds explicit `energy:` metadata for that same Item
- **THEN** the site keeps exactly one participant, the metadata wins wherever the two
  disagree, and no error is raised

> Source: surfaced by the wave-1 prototype (B9 — see docs/PROTOTYPE_FEEDBACK.md) — it is
> required by _Runtime contribution_, which admits add-on and script contributors on equal
> terms without saying what happens when two of them describe the same device, and by
> _Multiple contributors, user selection_, which says what may coexist but not what the
> engine does when two contributions collide; the corpus's only precedence statement,
> `participant-discovery` _Explicit declaration wins_, settled an explicit declaration
> against a **discovered** proposal in wave 4 and said nothing about a **contributed** one;
> not stated in the thread. Identity, duplicate handling and the precedence chain decided
> by the owner 2026-08-02 (D5, docs/OWNER_DECISIONS.md) — alternatives preserved in
> design.md §1 and §5.13; the discovery-side statement is now the same rule seen from the
> other side.

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

### Requirement: Engine-owned prohibitions are closed to contributors

A contributed algorithm, provider or script SHALL NOT lift an engine-owned prohibition —
the per-consumer `never` hands-off flag, a user-declared level gate, the readiness
interlock, device-protection constraints, or the enumeration of readings that count as
safety inputs — while the engine's own level-derived steering remains an ordinary
algorithm input that a contributor may override.

#### Scenario: A script cannot start a hands-off device

- **GIVEN** a consumer the user declared `never`
- **WHEN** a contributed algorithm returns a decision that would start it
- **THEN** the device is not written, and the decision is reported with its reason

#### Scenario: A planner may lift the engine's own gate

- **GIVEN** a consumer the engine's own level-derived steering would hold back and a
  deadline the contributed planner is working to
- **WHEN** the planner decides to run it now
- **THEN** the engine applies that decision, because its own steering is an input rather
  than a prohibition

#### Scenario: A contributor cannot promote its own reading to safety input

- **GIVEN** a contributed series — a forecast, say — whose provider declares it a safety
  input
- **WHEN** the engine builds its cycle snapshot
- **THEN** the claim is not honoured, the safety-input enumeration remaining the engine's,
  and the declaration is reported

> Source: surfaced by the wave-1 prototype as the security boundary the corpus left
> unstated (see docs/PROTOTYPE_FEEDBACK.md); decided by the owner 2026-08-02 (D13,
> docs/OWNER_DECISIONS.md) — prohibitions engine-owned, preferences algorithm inputs;
> alternatives preserved in design.md §6 (everything an algorithm input; everything
> engine-owned). Grounded in the reference binding, where gating lives in
> the asset handler below every controller, "the last line before an item is written". The
> same decision lifts `never` off the Simple class onto the consumer so it applies to all
> four classes — that requirement lives in `define-participant-model`, and its
> profile-selection consequence in `device-profiles`. Not stated in the thread.

### Requirement: A declared Item name that does not resolve is a runtime condition

A declaration naming an Item that does not exist SHALL be accepted when it is read and
treated at evaluation as a degraded source — reported as a warning on the engine's
observability surface, and bound without redeclaration if that Item appears later — rather
than rejected as a declaration in error.

#### Scenario: Metadata read before its Item

- **GIVEN** a file-based site whose `energy:` declaration is read before the Item it names
- **WHEN** the declaration is parsed
- **THEN** it is accepted, and it binds when the Item appears — the order in which the two
  arrive is not load-bearing

#### Scenario: The Item never appears

- **GIVEN** a participant declaring a measurement Item that no site configuration ever
  creates
- **WHEN** the engine evaluates
- **THEN** that reading counts as unavailable and the participant is reported as a degraded
  source at warning level, the engine continuing with what the declaration does provide —
  and where the missing reading is one the engine counts as a safety input, the safe state
  applies exactly as it does to a reading that has gone stale

#### Scenario: Unresolved is not malformed

- **GIVEN** one declaration whose bounds cannot be read and another whose bounds are fine
  but which names an Item that does not exist
- **WHEN** both are read
- **THEN** the first participant is skipped whole as a configuration error and the second
  is registered, because an unreadable declaration and an unresolved name are different
  failures with different remedies

> Source: surfaced by the wave-1 prototype (B15 — see docs/PROTOTYPE_FEEDBACK.md), which
> found nothing saying whether a declaration may name Items that do not exist, while the
> corpus's only neighbouring statement covers a measurement that disappears **after**
> resolving; core precedent — `ItemChannelLink` makes a forward reference legal and
> re-binds it later, which is why rejecting one would make declaration order matter on a
> file-based site; decided by the owner 2026-08-02 as a stated default (D17, Part B row
> EP-5.10, docs/OWNER_DECISIONS.md) — the alternative, treating it as a declaration error
> under `energy-participants` _Malformed declarations are reported, never partially
> accepted_, is preserved in design.md §5.10. Not stated in the thread.

### Requirement: Graceful degradation on contributor loss

When a contributor fails or is removed, the engine SHALL continue operating with the
remaining participants and any persisted baselines rather than halting or discarding
state.

#### Scenario: Forecast source removed mid-day

- **GIVEN** planning that uses a contributed solar forecast with a persisted baseline
- **WHEN** that add-on is uninstalled or its service goes dark
- **THEN** the engine keeps planning on the baseline and the remaining participants, and
  reports the degraded source on its observability surface — a deduplicated event and the
  engine's status Item

#### Scenario: Safety input lost — degrade safe, not blind

- **GIVEN** a live power measurement that feeds the electrical-limit floor, optionally
  carrying a declared maximum age
- **WHEN** it becomes unreadable, `UNDEF` or `NULL`, or exceeds that age
- **THEN** the engine does NOT keep optimizing on stale safety data — it refuses every
  increase, floors Controllable loads at their declared minimum, drops ModeControllable
  loads to their most-restricted mode and turns Simple loads off, all subject to device
  protections and never interrupting a running Batch programme, until the measurement
  returns

> Source: the baseline-fallback behaviour from mstormi — the EMS "to run well even if
> your internet connection or forecast provider fails for whatever reason, for whatever
> outage duration" ([5020830338](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5020830338)),
> extended here from data sources to any contributor; extension marked as design
> generalization. The safety-input distinction is **reference-production-sourced**
> (a staleness gate capping all loads when the live-measurement bridge goes silent) —
> data-plane loss degrades to baselines, safety-input loss degrades to safe. Staleness
> definition, freeze-and-floor safe state and the reporting surface decided by the owner
> 2026-08-02 (D14 and D16 pack A8, docs/OWNER_DECISIONS.md) — alternatives preserved in
> design.md §5.6 and §5.7; the normative runtime statement lives in
> `define-engine-contract`.

### Requirement: Core-shipped defaults without privilege

The framework SHALL allow generic default implementations to ship with the core itself —
the configurable grid-price provider being the named case — usable with zero add-ons
installed and replaceable by contributed alternatives on equal terms.

#### Scenario: Works out of the box, replaceable later

- **GIVEN** a fresh installation with no energy add-ons
- **WHEN** the user configures the core's generic grid-price provider
- **THEN** the EMS is functional, and later installing a dedicated price binding can take
  over that role without migration pain

#### Scenario: Equal terms, stated observably

- **GIVEN** core's default provider registered through the same SPI at
  `service.ranking = -2`
- **WHEN** a contributed alternative for that role registers at its own default ranking
- **THEN** the contributed one is used with no configuration change, and the core default
  is still selectable by naming it in the role's configuration

> Source: Kai — the generic `GridEnergyProvider`: "if we manage to implement this class
> in a configurable and generic way, we could possibly add this as a default
> implementation to the core directly"
> ([1483930195](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1483930195));
> the price-specific requirement lives in `price-data` — this states the architectural
> allowance; "equal terms" made observable by the owner 2026-08-02 (D18, Part C EP-5.12,
> docs/OWNER_DECISIONS.md) on core's `DefaultStateDescriptionFragmentProvider` precedent —
> the two readings not taken are preserved in design.md §5.12.
