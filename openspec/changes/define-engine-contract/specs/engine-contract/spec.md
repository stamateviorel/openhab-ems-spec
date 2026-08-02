# engine-contract

## ADDED Requirements

### Requirement: Central periodic evaluation

The system SHALL run one central engine that periodically evaluates all declared
participants — on a tick defaulting to 60 seconds plus a re-evaluation triggered by
changes to the readings participants declare, debounced by a default 5 seconds, both
defaults being configuration a site may change — against the same consistent context
snapshot, deriving the site energy level from that snapshot's own readings by calling the
level plane as a function rather than by reading a published Item, and produces the
cycle's control decisions.

#### Scenario: One snapshot per cycle

- **GIVEN** two consumers eligible for surplus in the same cycle
- **WHEN** the engine evaluates them
- **THEN** both are judged against the same surplus figure, not values read at different
  moments

#### Scenario: Level and readings cannot disagree

- **GIVEN** a cycle whose snapshot carries live PV surplus that escalates the site level
- **WHEN** an algorithm reads both the level and the surplus from that cycle
- **THEN** the level it sees is the one derived from that same surplus figure, so the two
  cannot disagree inside one cycle

#### Scenario: The published level Item is not the transport

- **GIVEN** an engine running shadow-only, writing no participant Item at all
- **WHEN** it evaluates a level-gated consumer
- **THEN** it still has a level to gate on, because the level reaches it in process and
  publication is an observable side effect rather than the path

#### Scenario: Provider data refreshed regularly

- **GIVEN** price providers publishing future prices
- **WHEN** the engine runs
- **THEN** it re-reads provider data on a regular cadence rather than requiring push
  notifications from providers

#### Scenario: A declared reading changes between ticks

- **GIVEN** an engine on the shipped defaults, 6 seconds after its last tick
- **WHEN** a reading a participant declares changes
- **THEN** a fresh cycle runs once the debounce interval has passed, rather than the
  change waiting up to a full tick to be noticed

#### Scenario: A burst of updates is one re-evaluation

- **GIVEN** a measurement source that publishes eight updates within two seconds
- **WHEN** the engine reacts to them
- **THEN** one re-evaluation runs after the debounce interval, not eight

#### Scenario: The cadence is a default, not a constant

- **GIVEN** a site that configures a 15-second tick and a 1-second debounce
- **WHEN** the engine runs
- **THEN** it evaluates on that cadence, nothing else in this contract changing with it

> Source: Kai's need #4 — "a central controller that mixes this all together and
> activates/controls the different devices"
> ([1481931209](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931209));
> lsiepel's Engine module ([1481931215](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931215));
> pull cadence — "the EMS regularly scrapes the prices from the providers — possibly up
> to every minute", push deferred
> ([1482078918](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1482078918));
> the direction of the level dependency (prototype I1) decided by the owner 2026-08-02
> (D6, docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §4 and §18; the tick,
> the event-responsive half and the debounce are stated **defaults** under the same owner's
> parameter pass (D17, Part B row EC-1, docs/OWNER_DECISIONS.md) — alternatives preserved
> in design.md §1.

### Requirement: Deterministic conflict resolution

When multiple decisions target the same device in one cycle — and several algorithms may
be active at once — the engine SHALL resolve the conflict deterministically in favour of
the better priority, which is the lower number, independent of registration or iteration
order.

#### Scenario: Better priority wins

- **GIVEN** a shedding decision carrying priority 10 and an optimization decision carrying
  priority 40, both addressing the same load
- **WHEN** the cycle is dispatched
- **THEN** the priority-10 decision is applied and the outcome is identical on every run
  with the same inputs

#### Scenario: Two algorithms, one device

- **GIVEN** a contributed surplus algorithm and the engine's device-protection algorithm
  both active and both addressing one consumer
- **WHEN** the cycle is dispatched
- **THEN** neither is disabled by the other's presence and the conflict is resolved by
  the same lower-number-wins rule that orders consumers

> Source: Kai's need #7 — "load selection must prioritize and weigh the different loads
> against each other" ([1481931209](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931209));
> deterministic ordering proven in the reference; decided by the owner 2026-08-02 (D4,
> docs/OWNER_DECISIONS.md) — alternatives preserved in
> `define-participant-model` design.md §5.2 and in design.md §19, which also record the
> known follow-on: the reference binding's `Controller` contract says "lower runs first,
> higher wins on conflict" and has to be reconciled with this.

### Requirement: Constraint precedence ladder

The engine SHALL resolve competing constraints by one fixed ladder — the master stop
above everything, then declared electrical limits, then device protections, then
user-declared level gates, then optimization — so that no configuration and no contributed
algorithm can reorder them, the ladder itself stating its only two exceptions: a Batch
programme already running and a consumer its owner marked hands-off, both of which the
electrical-limit rung books as load the others are trimmed against rather than sheds, the
floor refusing with a reason and reporting where the remainder still does not fit.

#### Scenario: The fuse outranks the freezer

- **GIVEN** a cooling device past its maximum OFF time and no headroom under the site's
  electrical limit
- **WHEN** the engine evaluates
- **THEN** the limit wins and the device stays off, and it is switched on as soon as
  headroom exists

#### Scenario: A protection outranks a gate

- **GIVEN** a consumer whose level gate is satisfied and whose minimum OFF time has not
  elapsed
- **WHEN** the engine evaluates
- **THEN** it stays off, the protection outranking the gate that would otherwise start it

#### Scenario: Optimization is bottom of the ladder

- **GIVEN** an algorithm proposing to switch off a load that is inside its minimum
  runtime, because the price just rose
- **WHEN** the cycle is dispatched
- **THEN** the load keeps running, an optimization decision never displacing a protection

#### Scenario: The two loads the electrical rung may not shed

- **GIVEN** an overload in which the only remaining candidates are a Batch programme
  mid-run and a consumer marked hands-off, everything else already at its minimum
- **WHEN** the floor resolves it
- **THEN** both are booked as load rather than shed, and the floor refuses with a reason
  reporting that the remainder does not fit — the exception being stated in the ladder
  rather than discovered by an implementer reading two requirements against each other

> Source: proposed in design.md §5 as the ladder the reference implements and never
> discussed in the thread; production precedent — `constrainOnOff`'s javadoc states it
> outright, "the fuse outranks the freezer, deliberately", and `applyBreakerGate`
> overrides every positive action regardless of origin; decided by the owner 2026-08-02
> (D2 for the ladder and D1 for the master stop's place above it, docs/OWNER_DECISIONS.md)
> — alternatives preserved in design.md §5 and §6. The two exceptions are **not** a
> weakening of D2 but the consequence of two other decisions landing on the same rung: D9
> exempts a running Batch programme (the taxonomy's non-interruptibility, which is a device
> protection outranking an electrical limit) and D13 lifts `never` onto every consumer
> class. Both are stated here rather than only in the requirements that create them, because
> a ladder that says "nothing may reorder these" and is then quietly reordered elsewhere is
> the defect this requirement exists to prevent — the reading in which **an electrical limit
> may interrupt a running Batch and may shed a hands-off load**, keeping the ladder
> absolute, is preserved in design.md §5 and §8 and in
> `define-participant-model` design.md §5.7.

### Requirement: Electrical limits outrank optimization

The engine SHALL enforce declared electrical limits — a total power budget and, where
declared, per-phase headroom — ahead of any optimization decision, resolving an overload
worst-priority-first over exactly four dispositions (trim, defer, leave untouched, refuse
with a reason) chosen by each load's own shape: a Controllable load trims to the largest
value at or above its declared minimum and otherwise defers, a Simple load and a Batch
programme that has not started defer — deferral of a Simple load that is already running
being a switch-off, subject to device protections, and replannable exactly as a deferred
start is — a Batch programme already running and a consumer marked hands-off are left
untouched with their draw re-booked as load the others are trimmed against, and a
ModeControllable load is exempt to the extent of the figure it declares.

#### Scenario: Budget respected at dispatch time

- **GIVEN** loads whose combined booked draw would exceed the declared budget this cycle
- **WHEN** the engine dispatches
- **THEN** the worst-priority loads are acted on first, each by the disposition its shape
  prescribes, until the total fits, regardless of what the plan proposed

#### Scenario: Shape decides how, priority decides who

- **GIVEN** an overload in which the worst-priority load is a plain switch that cannot be
  trimmed and a better-priority load is continuous
- **WHEN** the engine resolves the overload
- **THEN** the switch is deferred rather than trimmed, and being untrimmable neither
  moves it up nor down the order in which loads are acted on

#### Scenario: A running batch programme is not interrupted

- **GIVEN** an overload while a Batch programme is mid-run
- **WHEN** the engine resolves it
- **THEN** the programme is left untouched and its draw is re-booked as load the other
  participants are trimmed against, rather than being interrupted to make room

#### Scenario: A hands-off load is booked, never steered

- **GIVEN** an overload whose worst-priority load is an EV charger its owner marked
  hands-off
- **WHEN** the engine resolves it
- **THEN** that charger is booked as load the others are trimmed against and is neither
  trimmed, deferred nor switched off, exactly as a running Batch programme is

#### Scenario: Deferring a load that is already running

- **GIVEN** an overload and a running Simple load whose minimum runtime has elapsed
- **WHEN** the floor defers it
- **THEN** it is switched off, and the deferral is published as an outcome the planner may
  reschedule, so "defer" means the same thing for a running load as for one that has not
  started

#### Scenario: Phase-aware headroom

- **GIVEN** a consumer that declares which phase(s) it draws on
- **WHEN** one phase is near its limit
- **THEN** the engine accounts for that phase specifically, not only the site total,
  covering the load it dispatches itself plus whatever the providers declare per phase

#### Scenario: Overload resolution is not an implementation accident

- **GIVEN** an overload in which one load can be reduced to a smaller draw and another can
  only be switched off entirely
- **WHEN** the engine resolves the overload twice from the same inputs
- **THEN** the outcome for each load follows the stated dispositions and is the same both
  times

> Source: seime — "limit the total load within the hour (or 15 minute buckets)"
> ([1481931351](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931351));
> masipila — per-phase load balancing, "you can't toggle all the devices on at the same
> time" ([1481931297](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931297));
> relation: `grid-constraints` covers cost-optimal _planning_ under budgets — this is the
> runtime _floor_ beneath any plan. The "regardless of the active algorithm"
> generalization is a design synthesis from the reference's safety chain, **flagged for
> review**; sharpened after the wave-1 prototype surfaced E13 — "trimmed or deferred"
> never said which, when (see docs/PROTOTYPE_FEEDBACK.md); the rule itself decided by the
> owner 2026-08-02 (D9, with the four dispositions and the per-phase coverage clause from
> D16 / packs A6 and A10, docs/OWNER_DECISIONS.md) — alternatives preserved in design.md
> §8 and §20. Two clauses are reconciliations rather than decisions, and are marked as such:
> the hands-off exemption follows from D13 lifting `never` onto every consumer class
> (`define-participant-model` _Hands-off consumers_), which left this requirement
> dispositioning every load with no carve-out for it; and what deferral means for a load
> already running was never stated anywhere, while the floor books running loads — so it is
> made definite here as a switch-off rather than left for each implementation. Both readings
> the corpus previously allowed are preserved in design.md §8.

### Requirement: What the limit floor books

The floor SHALL book against every participant it constrains the larger of that
participant's declared figure and its live measurement while it is running, and its
declared figure while it is not, re-booking from scratch every cycle with no reserve
carried from one cycle to the next.

#### Scenario: A device that ignores its envelope

- **GIVEN** a running consumer commanded to 3 kW whose declared measurement reads 4 kW
- **WHEN** the floor books it
- **THEN** it books 4 kW, so the overshoot is constrained rather than hidden behind the
  command

#### Scenario: Admitting a load that is not running

- **GIVEN** a consumer that is off, whose measurement therefore reads zero
- **WHEN** the floor considers admitting it
- **THEN** it books the declared figure, so the room to start the load is reserved

#### Scenario: No reserve is carried

- **GIVEN** a cycle in which the previous cycle's booking proved too low
- **WHEN** the floor evaluates the same inputs again
- **THEN** it books them from scratch and reaches the same result, no error term from an
  earlier cycle changing the outcome

> Source: surfaced by the wave-1 prototype (E14, docs/PROTOTYPE_FEEDBACK.md) — the floor
> allocates against draw that has not happened yet, and nothing said which quantity it
> books nor how the error is reconciled; production precedent — the reference's taxonomy
> states "the planner must plan on measured power, not commanded values", and stability
> comes from smoothing the input rather than from a reserve; decided by the owner
> 2026-08-02 (D15, docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §9.

### Requirement: Safe state on a degraded safety input

When a reading the engine counts as a safety input is stale, the engine SHALL freeze and
floor — refusing every increase, flooring Controllable loads at their declared minimum,
dropping a ModeControllable load to its most restricted mode and switching Simple loads
off — subject to device protections, and never touching a Batch programme that is already
running or a consumer its owner marked hands-off.

#### Scenario: The measurement bridge goes silent

- **GIVEN** a site whose live power measurement stops updating
- **WHEN** the engine evaluates
- **THEN** no load is increased and the Controllable loads are held at their declared
  minimum rather than at whatever they were last allowed

#### Scenario: A protection holds a load the safe state would shed

- **GIVEN** a Simple load inside its minimum runtime when a safety input goes stale
- **WHEN** the engine applies the safe state
- **THEN** the load is held until the protection permits, the safe state refusing
  increases immediately but shedding only when the ladder allows

#### Scenario: A running batch is not interrupted

- **GIVEN** a Batch programme mid-run when a safety input goes stale
- **WHEN** the engine applies the safe state
- **THEN** the programme runs to completion and only other participants are frozen and
  floored

#### Scenario: A hands-off load is not shed by the safe state either

- **GIVEN** a Simple load marked hands-off when a safety input goes stale
- **WHEN** the engine applies the safe state
- **THEN** it is not switched off, its draw is still counted, and the freeze applies to
  every load the engine does steer

#### Scenario: Recovery

- **GIVEN** an engine in the safe state
- **WHEN** the stale reading updates again and is fresh by the participant's declared
  maximum age
- **THEN** normal operation resumes without the user clearing anything by hand

> Source: makes definite the "floor levels or pause" of `extension-surface`'s _Graceful
> degradation on contributor loss_ ("Safety input lost — degrade safe, not blind"), which
> the wave-1 prototype reported as inert because neither staleness nor the safe state had
> an operational meaning (E12, docs/PROTOTYPE_FEEDBACK.md); production precedent — the
> reference's building caps every charger at its 6 A minimum when the measurement bridge
> looks dead, rather than pausing the site; decided by the owner 2026-08-02 (D14,
> docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §21. The hands-off
> exemption is a reconciliation, not part of D14: D13 lifted `never` onto every consumer
> class after this requirement was written, and a safe state that switches a hands-off load
> off would make the flag advisory in exactly the situation a user relies on it — the
> reading in which the safe state may shed a hands-off load is preserved in
> `define-participant-model` design.md §5.7. **One consequence a maintainer should see
> before building this:** dropping a ModeControllable load to its most restricted mode
> means SG-ready mode 1, whose BWP cap — at most two hours per assertion and three
> assertions per day — `define-energy-levels` design.md §3 records as real, quantified,
> participant-side, and **not expressible in the participant model**. A long staleness
> therefore parks an SG-ready heat pump in a mode with a cap nothing in this corpus
> enforces.

### Requirement: Protection timing comes from device state history

The engine SHALL measure every protection duration from the participant Item's own last
state change and keep no protection timers of its own, starting the clock at first
observation where no last state change is available and treating that participant as
protection-unknown until one is.

#### Scenario: A cooldown survives a restart

- **GIVEN** a compressor whose Item is persisted with restore-on-startup and which went
  OFF ten minutes into a thirty-minute cooldown
- **WHEN** openHAB restarts and the engine evaluates
- **THEN** the remaining twenty minutes are honoured, derived from the Item's last state
  change rather than from anything the engine remembered

#### Scenario: No history yet

- **GIVEN** a protected participant whose Item reports no last state change
- **WHEN** the engine evaluates it
- **THEN** it starts the clock at that first observation and reports the participant as
  protection-unknown, rather than assuming the protection has elapsed

#### Scenario: The engine holds no timers

- **GIVEN** an engine that was stopped for an hour
- **WHEN** it resumes and evaluates a protected participant
- **THEN** the elapsed time is what the Item's history says it is, the gap in the engine's
  own operation changing nothing

> Source: surfaced by the wave-1 prototype (N3, docs/PROTOTYPE_FEEDBACK.md) — the four
> protection durations are all measured from the last state change and no requirement named
> the source, which decides whether a compressor's cooldown survives a restart; core
> precedent — `Item.getLastStateChange()` exists and `PersistenceManagerImpl.restoreItemStateOnStartup`
> restores it explicitly; decided by the owner 2026-08-02 (D7 and its refinement,
> docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §4.

### Requirement: An unpersisted protection is reported

The engine SHALL report every participant that carries a device protection whose Item
state is not persisted across a restart, so that a guarantee degraded by the site's
persistence configuration is visible rather than assumed.

#### Scenario: Protections declared, persistence absent

- **GIVEN** a fridge declaring a maximum OFF time on an Item that no persistence service
  restores on startup
- **WHEN** the engine evaluates it
- **THEN** it reports that participant as one whose protection will not survive a restart,
  and continues to steer it

#### Scenario: The report clears

- **GIVEN** a participant reported for an unpersisted protection
- **WHEN** the site configures persistence with restore-on-startup for that Item
- **THEN** the report clears on its own, without the participant being redeclared

> Source: required by the mechanism chosen for _Protection timing comes from device state
> history_ — core restores `lastStateChange` only for Items persisted with
> `restoreOnStartup`, so the guarantee is contingent on the site's own configuration and
> the contingency has to be visible; a departure from the reference binding, whose
> `constrainOnOff` leaves the desired state untouched when the elapsed time is unknown;
> decided by the owner 2026-08-02 (D7 refinement, docs/OWNER_DECISIONS.md) — alternatives
> preserved in design.md §4.

### Requirement: Shadow mode

The engine SHALL provide an observe-only shadow mode — computing and logging every
decision without writing to any participant Item, engageable globally and narrowable to a
single algorithm or a single participant — and default to it on first install with the
master stop disengaged, shadow and the stop being two separate controls.

#### Scenario: Fresh install writes nothing

- **GIVEN** a newly configured installation
- **WHEN** the engine runs its first cycles
- **THEN** decisions appear as shadowed outcomes and no device is actuated until the user
  leaves shadow mode

#### Scenario: Fresh install is shadowed, not stopped

- **GIVEN** a newly configured installation
- **WHEN** the user inspects the two controls
- **THEN** shadow is engaged and the master stop is disengaged, so the engine evaluates
  and reports from the first cycle

#### Scenario: One participant held back

- **GIVEN** an engine running live with one participant placed in shadow
- **WHEN** the cycle is dispatched
- **THEN** every other participant is actuated and that one's decisions are reported as
  shadowed without being written

#### Scenario: One algorithm held back

- **GIVEN** an engine running live with a newly contributed algorithm placed in shadow
  while the others dispatch
- **WHEN** the cycle runs
- **THEN** that algorithm's decisions are reported as shadowed and never written, the
  others are actuated, and a contributed algorithm can therefore be evaluated on a live
  site without a site-wide shadow

#### Scenario: Side-by-side validation

- **GIVEN** an existing user automation still in control
- **WHEN** the engine runs in shadow next to it
- **THEN** its would-have-done decisions can be compared against the existing behaviour
  before any cutover

> Source: shadow-first validation as practiced and reported in-thread — unit tests, then
> "weeks of shadow mode next to my old production rules comparing every decision, and
> only then live control"
> ([5014707411](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5014707411));
> reference default: shadow TRUE on first install; the two-control shape, the granularity
> and the fresh-install state decided by the owner 2026-08-02 (D3,
> docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §2.

### Requirement: Master stop

A single master control SHALL halt the engine entirely when engaged — no evaluation, no
contributed algorithm invoked, no write — without requiring uninstallation or
reconfiguration of participants, and above every other precedence including device
protections, whose non-enforcement is reported for as long as it is engaged.

#### Scenario: Stop mid-plan

- **GIVEN** the engine actively steering devices
- **WHEN** the master control is engaged
- **THEN** writes cease immediately, and disengaging it resumes normal operation

#### Scenario: The stop is where `stopped` comes from

- **GIVEN** a cycle in flight when the master control is engaged
- **WHEN** the stop takes effect
- **THEN** the decisions that cycle had not yet dispatched are published with the outcome
  `stopped` and none of them is written, and no later cycle produces a decision at all
  while the stop is engaged

#### Scenario: Nothing contributed runs while stopped

- **GIVEN** a contributed algorithm that holds its own access to openHAB
- **WHEN** the master control is engaged
- **THEN** the engine does not invoke it at all, so "stopped" stops the site's writes and
  not merely the engine's own

#### Scenario: Resume starts from a clean slate for what the engine held

- **GIVEN** a device commanded moments before the stop was engaged, with its
  acknowledgement window still open
- **WHEN** the stop is disengaged an hour later
- **THEN** the engine resumes with that window cleared and no ephemeral state of its own
  carried across the gap, while every device protection is unaffected because its elapsed
  time is read from the device's own history rather than from the engine

#### Scenario: Protections are not enforced, and that is said

- **GIVEN** a cooling device relying on the engine for its duty-cycle guarantee
- **WHEN** the master control is engaged
- **THEN** the guarantee is not enforced and the engine reports that it is not, rather
  than leaving an operator to infer that device safety reverted to whoever owned it
  before the EMS

> Source: **reference production experience** (master shadow flag as the documented kill
> switch, default-safe) — not stated in the thread; flagged as such; what the stop halts
> decided by the owner 2026-08-02 (D1, docs/OWNER_DECISIONS.md), **overriding the
> recommendation put to them**, which was dispatch-only — alternatives preserved with both
> arguments in design.md §6, and the stop's place above the ladder in §5. What survives a
> stop is the same owner's stated default (D17, Part B row EC-7): engine-held ephemeral
> state resumes cleared, protections continue by construction because they are read from
> device state history — alternatives preserved in design.md §7, which also records the two
> state items those decisions leave open. The `stopped` outcome of _Decision outcomes are
> named and published_ is reachable only here, and only for a cycle already in flight: once
> the stop is engaged no later cycle evaluates, so nothing later can produce a decision to
> mark. The alternative — abandon the in-flight cycle unpublished and drop `stopped` from
> the per-decision vocabulary entirely — is preserved in design.md §12.

### Requirement: Acknowledgement-aware actuation

The engine SHALL avoid re-sending a command to a device while the previous command is
still unacknowledged, within an acknowledgement window of 60 seconds by default and
overridable per participant, an acknowledgement being the control item's state matching
the commanded value — exactly, under a unit-aware comparison, or within a tolerance band
where the participant declares one — and the window's expiry lapsing the outstanding
command so that actuation for that device resumes.

#### Scenario: Charger with lagging state

- **GIVEN** an EV charger whose current-limit item does not reflect a command until the
  device acknowledges it
- **WHEN** the engine has sent a new limit and the state still shows the old value
- **THEN** the command is not re-sent every cycle while the acknowledgement window is
  open

#### Scenario: Device that never acknowledges

- **GIVEN** a device that never reflects a commanded value in its reported state
- **WHEN** its acknowledgement window elapses
- **THEN** the outstanding command lapses, the participant is reported as unacknowledged,
  and the engine may command it again — it is not withheld from management by default

#### Scenario: A reading inside the tolerance band

- **GIVEN** a wallbox commanded to 16 A which reports 15.999 A, and a declared tolerance
  band that covers the difference
- **WHEN** the engine tests for acknowledgement
- **THEN** the command counts as acknowledged, whereas with no band declared it does not

#### Scenario: A participant that needs longer than the default

- **GIVEN** one device declaring a three-minute acknowledgement window on an engine whose
  default is 60 seconds
- **WHEN** both that device and an undeclared one are commanded and neither echoes
- **THEN** the undeclared one's command lapses after 60 seconds and the declaring one's
  after three minutes, the override applying to that participant alone

> Source: **reference production experience** — an ACK window preventing command
> re-sends, essential for `autoupdate="false"` OCPP devices; not stated in the thread;
> flagged as such; sharpened after the wave-1 prototype surfaced E8 — "unacknowledged"
> was never made operational (see docs/PROTOTYPE_FEEDBACK.md); what counts as an
> acknowledgement and what expiry does decided by the owner 2026-08-02 (D8), the window's
> 60-second default under the parameter pass (D17, row EC-3a, docs/OWNER_DECISIONS.md) —
> alternatives preserved in design.md §3, which also records what those decisions did
> **not** settle: where acknowledgement handling lives, the default tolerance value, what
> happens to a changed command, and how this interacts with a reduction the electrical-limit
> floor needs to make while a command is outstanding. What a participant may declare —
> the window and the band, the band being absolute in the control Item's dimension — is
> stated once in `energy-participants` _Acknowledgement declaration_ rather than implied
> here.

### Requirement: Replaceable algorithm, scripts first-class

The optimization algorithm SHALL be replaceable by the user — implementable as scripts or
rule templates, not only as compiled code — while one complete, closed enumeration stays
engine-owned for every algorithm: evaluation, the electrical-limit floor, shadow, the
master stop, actuation, and the user's prohibitions, namely the hands-off flag, the
user-declared level gate, the readiness interlock, device protections and which readings
count as safety inputs — the engine's own level-derived steering being the one member an
algorithm may override.

#### Scenario: Script algorithm under the same guardrails

- **GIVEN** a user replaces the default algorithm with their own script
- **WHEN** the script proposes decisions
- **THEN** they flow through the same limit enforcement, shadow gating and actuation
  semantics as the default's

#### Scenario: An enumerated responsibility cannot be taken over

- **GIVEN** a contributed algorithm that writes to a device directly instead of returning
  a decision
- **WHEN** it runs
- **THEN** that write does not reach the device, because actuation is a member of the
  stated engine-owned enumeration

#### Scenario: A prohibition is not an input

- **GIVEN** a consumer its owner marked hands-off and a contributed algorithm proposing to
  start it
- **WHEN** the cycle is dispatched
- **THEN** the device is not started, and the algorithm cannot reach the flag to clear it

#### Scenario: The engine's own steering is overridable

- **GIVEN** a contributed planner that needs a consumer to run now to meet a declared
  deadline, while the engine's own level-derived steering would hold it back
- **WHEN** the planner proposes to run it and no user prohibition applies
- **THEN** the decision stands, the engine's own steering being an input rather than a
  prohibition

> Source: Kai — the EMS "could possibly then even be as simple as a rule template, so
> that people could e.g. implement easily alternative energy management algorithms and
> also do this through scripts"
> ([1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374));
> the guardrail tail follows from the engine-owned requirements above; sharpened after the
> wave-1 prototype surfaced N7 — the enumeration was stated once and not stated to be
> complete (see docs/PROTOTYPE_FEEDBACK.md); its membership, and the line between a
> prohibition and a preference, decided by the owner 2026-08-02 (D13,
> docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §16. This is the security
> boundary of the extension surface.

### Requirement: Decision outcomes are named and published

Every decision the engine produces SHALL carry one outcome from a fixed vocabulary —
applied, shadowed, stopped, superseded, deferred, suppressed, withheld, rejected —
together with a free-text reason, published as events that are deduplicated so an
unchanged decision is not re-emitted, readable for the current cycle over REST, and
summarised on one engine-published status Item, with no Item written for an individual
decision.

#### Scenario: A losing decision is named, not silent

- **GIVEN** two decisions addressing one device, one of which loses on priority
- **WHEN** the cycle is dispatched
- **THEN** the losing decision is published as superseded with its reason, rather than
  disappearing

#### Scenario: A decision naming an unknown participant

- **GIVEN** a contributed script that addresses a participant that does not exist
- **WHEN** it runs
- **THEN** the decision is rejected with a reason and counted, so a typo'd script does not
  look like a working one that never fires

#### Scenario: Shadow output is comparable

- **GIVEN** an engine running in shadow beside an existing user automation
- **WHEN** it evaluates over a week
- **THEN** each decision is published as shadowed with its reason and re-published only
  when it changes, so the comparison is made against events rather than against a log line
  repeated every cycle

#### Scenario: Shadow still writes no device

- **GIVEN** an engine in shadow mode publishing outcomes
- **WHEN** a decision is produced for a participant
- **THEN** no participant Item is written, the outcome travelling as an event and the
  engine's own status Item being the only Item it maintains

> Source: surfaced by the wave-1 prototype (N6, E15, E18, docs/PROTOTYPE_FEEDBACK.md) —
> _Shadow mode_'s own "Side-by-side validation" scenario asks a user to compare decisions
> the corpus had no vocabulary for and no surface to carry; core precedent — the
> `RuleStatus` / `RuleStatusDetail` / `RuleStatusInfoEvent` trio plus a REST resource is
> core's own shape for exactly this, and core publishes events while leaving persistence to
> whoever wants it;
> production precedent — `SetpointRequest.reason` is already a short human phrase used for
> both the shadow line and the last-action channel; decided by the owner 2026-08-02 (D16 /
> pack A8, docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §12, §13 and §14.

### Requirement: Participant conditions are reported on one surface

Every participant-level condition these requirements name — a declaration gap, a
participant whose protection elapsed time is unknown, an unacknowledged command, a
declaration naming an Item that does not resolve, and a protection whose state is not
persisted — SHALL be reported on one surface as a configuration-status message keyed on the
Item carrying the declaration and summarised on the engine's status Item, clearing on its
own when the condition ends and never requiring the participant to be redeclared.

#### Scenario: A declaration gap appears and then clears

- **GIVEN** a Simple consumer declaring an on-threshold and no `ratedPower`, so the floor
  books a borrowed figure
- **WHEN** the engine registers it, and later the user declares a `ratedPower`
- **THEN** the gap is reported against that consumer's own Item while it lasts and clears
  by itself once the rating is declared, the participant never being redeclared

#### Scenario: One surface, not five

- **GIVEN** a participant that is simultaneously protection-unknown and carrying an
  unacknowledged command
- **WHEN** a UI or a rule asks the framework what is wrong with it
- **THEN** both conditions are found in the same place and in the same vocabulary as the
  malformed-declaration reports of `energy-participants` _Malformed declarations are
  reported, never partially accepted_, rather than one in a log line and one in an event

#### Scenario: A reported condition is not a rejection

- **GIVEN** a participant carrying a reported condition
- **WHEN** the engine evaluates the cycle
- **THEN** it continues to steer that participant under the requirement that raised the
  condition, the report making a degraded guarantee visible rather than withdrawing the
  participant from management

> Source: required by the five requirements that raise these conditions and by none of
> which a surface was named — `energy-participants` _Consumer power figure_ and _Phase
> declaration_ (declaration gaps), _Protection timing comes from device state history_
> (protection-unknown), _Acknowledgement-aware actuation_ (unacknowledged), _An unpersisted
> protection is reported_, and `extension-surface` _A declared Item name that does not
> resolve is a runtime condition_; core precedent — `ConfigStatusProvider` with
> `ConfigStatusMessage{INFORMATION,WARNING,ERROR}` is already the surface
> `energy-participants` _Malformed declarations are reported, never partially accepted_
> uses, so these conditions join it rather than inventing a second one; decided by the owner
> 2026-08-02 (D16 / pack A8, which states that "a lost or degraded contributor, a stale
> safety input, and a protected participant whose state is not persisted all report here
> too", docs/OWNER_DECISIONS.md) — alternatives preserved in design.md §12 and §13. What A8
> defined and this makes definite are different objects: the outcome vocabulary of _Decision
> outcomes are named and published_ describes what became of a **decision**, while these
> conditions belong to a **registered, steered participant** whose declaration is
> incomplete — neither a decision nor a malformed declaration, and so previously homeless.
