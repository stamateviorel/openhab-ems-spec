# participant-discovery

## ADDED Requirements

### Requirement: Propose participants from the semantic model

The system SHALL derive a proposed set of energy participants from openHAB's semantic
model — mapping the **leaf** Equipment tag, Point types and Property tags to provider roles
and consumer classes — so a semantically modelled instance can be set up without
hand-declaring each item.

#### Scenario: PV provider from an inverter

- **GIVEN** an Equipment tagged `Inverter` (or `SolarPanel`) containing a `Measurement`
  Point with the `Power` property
- **WHEN** discovery runs
- **THEN** it proposes that item as a PV provider

#### Scenario: Controllable battery from equipment + setpoint

- **GIVEN** an Equipment tagged `Battery` with a `Power` `Measurement` and a `Setpoint`
  Point
- **WHEN** discovery runs
- **THEN** it proposes a controllable battery provider whose setpoint is that Point

#### Scenario: Consumer class from equipment kind

- **GIVEN** a `Dishwasher` with a `Switch` Point, a `HeatPump` with a `Mode` Point, and a
  `Boiler` with a `Switch` Point
- **WHEN** discovery runs
- **THEN** it proposes the dishwasher as Batch, the heat pump as ModeControllable, and the
  boiler as Simple

#### Scenario: The leaf tag decides, not its parent

- **GIVEN** an `EVSE` and a `UPS`, both of which sit under `PowerSupply` in the semantic
  tag tree
- **WHEN** discovery runs
- **THEN** the EVSE is proposed as a consumer and the UPS is proposed as nothing, no rule
  reading the parent tag as the role

#### Scenario: EVSE control granularity read, not assumed

- **GIVEN** an `EVSE` whose setpoint Point carries the `Current` property and a
  `StateDescription` with min and max
- **WHEN** discovery runs
- **THEN** it proposes an amps-denominated controllable consumer with that range, and a
  setpoint carrying neither a Property tag nor a UoM dimension is put to the user instead
  of being assumed to be 32 A or 11 kW

> Source: openHAB semantic model — `openhab-core` `SemanticTags.csv`: Equipment
> `PowerSupply`→`Battery`/`Inverter`/`SolarPanel`/`EVSE`/`Generator`, `Sensor`→`ElectricMeter`,
> `HVAC`→`HeatPump`/`Boiler`/`WaterHeater`, `WhiteGood`→`Dishwasher`/`WashingMachine`/`Freezer`;
> Points `Measurement`/`Setpoint`/`Switch`/`Control`/`Forecast`; Properties
> `Power`/`Energy`/`Current`/`Voltage`. Intent from Kai's metadata-marking + out-of-the-box
> UI ([1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374),
> [1481931227](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931227)).
> New proposal — see the proposal.md provenance note. The leaf-tag rule and the EVSE
> granularity rule were decided by the owner 2026-08-02 (D17, Part B DP-1(ii) and the
> DP-1 rule-shape warning, docs/OWNER_DECISIONS.md); the design note's "read from
> UoM/state-description" is corrected there, since `Current` and `Power` are sibling
> first-class Property tags. Alternatives preserved in design.md §1.

### Requirement: The grid meter is a candidate, never an auto-assignment

Where the model carries electric meters, discovery SHALL propose every `ElectricMeter`
with a Power measurement as a ranked candidate for the grid connection — one directly
under a Location ranking first — and never assign the role itself, even when exactly one
candidate exists.

#### Scenario: Seven meters, no guess

- **GIVEN** a site whose model contains several `ElectricMeter` equipment items with Power
  measurements
- **WHEN** discovery runs
- **THEN** all of them are offered as candidates in that ranking, and none is proposed as
  the grid connection until the user picks one

#### Scenario: A single candidate is still a question

- **GIVEN** a site with exactly one `ElectricMeter`
- **WHEN** discovery runs
- **THEN** it is offered as the leading candidate and still requires confirmation, because
  the model carries no grid tag to make it more than a guess

> Source: decided by the owner 2026-08-02 (D17, Part B DP-1(i), docs/OWNER_DECISIONS.md) —
> `SemanticTags.csv` has no grid tag and no top-level/sub-meter distinction, and the owner's
> own site has seven indistinguishable clamp meters on one bridge; the auto-assign-if-single
> alternative is preserved in design.md §1. Extends this change's own propose-and-confirm
> guardrail; not stated in the thread.

### Requirement: Propose, never silently activate

Discovery SHALL present its inferences as a reviewable proposal and never steer a device
from an inferred configuration until the user accepts it.

#### Scenario: Nothing moves before confirmation

- **GIVEN** discovery has proposed a full participant set
- **WHEN** the user has not yet accepted it
- **THEN** the engine actuates nothing from the proposal

> Source: the shadow-first validation culture of the thread
> ([5014707411](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5014707411))
> and the wave-4 "propose, don't override" guardrail; safety design.

### Requirement: Proposals live in a reviewable registry

Proposals SHALL be held in an EMS-owned registry whose lifecycle mirrors the Inbox —
new, ignored, or approved into a declaration — keyed on participant identity, with
approval writing the `energy:` metadata the user would otherwise have hand-written.

#### Scenario: Ignored stays ignored

- **GIVEN** a proposal the user marked ignored
- **WHEN** discovery runs again over an unchanged model
- **THEN** it is not offered again, and no declaration exists for it

#### Scenario: Approval writes what a user would have written

- **GIVEN** a proposal the user approves in the setup surface
- **WHEN** approval completes
- **THEN** the result is `energy:` metadata on the item, indistinguishable from a
  hand-written declaration, and the proposal leaves the registry

> Source: decided by the owner 2026-08-02 (D17, Part B DP-2, docs/OWNER_DECISIONS.md) —
> an EMS-owned registry rather than core's Inbox, because `DiscoveryResult.getThingUID()`
> and `Inbox.add`'s dedupe-on-Thing-ID mean an item-keyed proposal cannot reuse the Inbox
> without a fake `ThingUID`, while the lifecycle users already know is still the right one;
> identity is the Item name (D5). The draft-metadata alternative is preserved in
> design.md §2. The review surface itself is `define-energy-ui` _Guided setup surface_. Not stated in
> the thread.

### Requirement: Intent stays user-owned, conventions are fixed centrally

Demand and deadlines, flexibility and priority, and protection limits SHALL remain
user-supplied or learned and never inferred from the model, while the sign convention is
not asked at all — it is fixed by the specification, and what a discovered device needs
from the user is only whether its reading follows that convention.

#### Scenario: The convention is stated, the device's polarity is confirmed

- **GIVEN** an `ElectricMeter` with a `Power` `Measurement` discovered as a grid candidate
- **WHEN** the proposal is presented
- **THEN** the specification's convention (grid + = export, − = import) is stated rather
  than asked, and the user confirms only whether this device reports that way or needs the
  invert flag

#### Scenario: Intent is never inferred

- **GIVEN** a discovered dishwasher
- **WHEN** the proposal is presented
- **THEN** its demand, deadline and priority are left for the user or the learning layer,
  the model carrying none of them

> Source: sign convention is not in the semantic model — it needs the invert-flag
> normalization the reference binding documents in its capability layer; demand,
> flexibility and protections are user intent (design); learning fills them over time
> (wave 4). Reworded following the owner's 2026-08-02 decision that sign conventions are
> fixed centrally for every role and setpoint, with disagreeing devices normalised at the
> edge (D11, docs/OWNER_DECISIONS.md) — so the convention is no longer a user choice, only
> the per-device invert flag is; the per-participant-convention alternative is preserved
> with the normative requirement in `define-participant-model`.

### Requirement: Explicit declaration wins

An explicit `energy:` declaration SHALL take precedence over any discovered proposal for
the same participant — identified by the Item name, or by an explicit `id` overriding it —
as the last link of the one precedence chain the corpus uses, explicit metadata over
contributed over discovered.

#### Scenario: Hand-tuned item keeps its declaration

- **GIVEN** an item carrying an explicit `energy:` consumer declaration
- **WHEN** discovery would infer a different class for it
- **THEN** the explicit declaration is kept and the inference is discarded

#### Scenario: A contributed declaration also outranks a proposal

- **GIVEN** a participant an add-on contributed and a discovery proposal for the same Item
- **WHEN** both are present
- **THEN** the contributed declaration stands and the proposal is not offered, the chain
  ranking a discovered proposal below both other kinds

> Source: declaration is authoritative in the participant model; metadata-wins merge
> behaviour proven in the reference (`MetadataParticipantScanner`); re-keyed from "the same
> item" to participant identity and folded into the single precedence chain by the owner's
> decision of 2026-08-02 (D5, docs/OWNER_DECISIONS.md) — alternatives preserved in
> `define-extension-points` design.md §1 and in design.md §5, which asked whether the two
> precedence statements are one rule or two: they are one.

### Requirement: No guessing without semantics

Where semantic tags are absent for an item, discovery SHALL propose nothing for it rather
than guess from names, so an unmodelled instance yields an empty proposal, not a wrong
one.

#### Scenario: Untagged power item is left alone

- **GIVEN** a `Number:Power` item with no semantic tags
- **WHEN** discovery runs
- **THEN** it is not proposed as a participant

#### Scenario: Equipment the role model cannot place is parked

- **GIVEN** equipment tagged `Generator`, `WindGenerator` or `UPS`
- **WHEN** discovery runs
- **THEN** nothing is proposed for it unless the user opts in, the participant model's
  roles — grid, PV and battery on the provider side, consumer on the other — naming none
  these fit

> Source: design decision to avoid fragile name-heuristics; the honest limit that
> discovery is only as good as the modelling (owner design discussion, 2026-07-20). The
> parked-equipment scenario was decided by the owner 2026-08-02 (D17, Part B DP-1(iii),
> docs/OWNER_DECISIONS.md) — alternative preserved in design.md §1: propose them as
> providers once the provider-role vocabulary grows.
