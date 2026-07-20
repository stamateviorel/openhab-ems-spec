# participant-discovery

## ADDED Requirements

### Requirement: Propose participants from the semantic model
The system SHALL derive a proposed set of energy participants from openHAB's semantic
model — mapping Equipment tags, Point types and Property tags to provider roles and
consumer classes — so a semantically modelled instance can be set up without
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

> Source: openHAB semantic model — `openhab-core` `SemanticTags.csv`: Equipment
> `PowerSupply`→`Battery`/`Inverter`/`SolarPanel`/`EVSE`/`Generator`, `Sensor`→`ElectricMeter`,
> `HVAC`→`HeatPump`/`Boiler`/`WaterHeater`, `WhiteGood`→`Dishwasher`/`WashingMachine`/`Freezer`;
> Points `Measurement`/`Setpoint`/`Switch`/`Control`/`Forecast`; Properties
> `Power`/`Energy`/`Current`/`Voltage`. Intent from Kai's metadata-marking + out-of-the-box
> UI ([1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374),
> [1481931227](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931227)).
> New proposal — see the proposal.md provenance note.

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

### Requirement: Intent and conventions stay user-owned
Sign convention (grid export vs. import), demand and deadlines, flexibility and priority,
and protection limits SHALL remain user-supplied or learned, never inferred from the
model, because the model does not carry them.

#### Scenario: Grid sign is confirmed, not guessed
- **GIVEN** an `ElectricMeter` with a `Power` `Measurement` discovered as the grid
- **WHEN** the proposal is presented
- **THEN** the + = export / − = import convention is asked or confirmed, not inferred

> Source: sign convention is not in the semantic model — it needs the invert-flag
> normalization the reference binding documents in its capability layer; demand,
> flexibility and protections are user intent (design); learning fills them over time
> (wave 4).

### Requirement: Explicit declaration wins
An explicit `energy:` declaration on an item SHALL take precedence over any discovered
proposal for the same item, so discovery seeds configuration without overriding
hand-tuning.

#### Scenario: Hand-tuned item keeps its declaration
- **GIVEN** an item carrying an explicit `energy:` consumer declaration
- **WHEN** discovery would infer a different class for it
- **THEN** the explicit declaration is kept and the inference is discarded

> Source: declaration is authoritative in the participant model; metadata-wins merge
> behaviour proven in the reference (`MetadataParticipantScanner`).

### Requirement: No guessing without semantics
Where semantic tags are absent for an item, discovery SHALL propose nothing for it rather
than guess from names, so an unmodelled instance yields an empty proposal, not a wrong
one.

#### Scenario: Untagged power item is left alone
- **GIVEN** a `Number:Power` item with no semantic tags
- **WHEN** discovery runs
- **THEN** it is not proposed as a participant

> Source: design decision to avoid fragile name-heuristics; the honest limit that
> discovery is only as good as the modelling (owner design discussion, 2026-07-20).
