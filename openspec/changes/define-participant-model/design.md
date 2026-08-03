# Design notes — open questions first

These are the contested decisions. Where one is still open it is deliberately **not**
encoded in the requirements; it needs maintainer alignment before any implementation change
is proposed.

**Answered questions carry an ANSWERED block.** On 2026-08-02 the owner of the reference
implementation answered most of them (`docs/OWNER_DECISIONS.md`). Those are **owner
decisions in a reference implementation, not thread consensus**, and every requirement they
changed says so on its Source line. The options that were _not_ chosen stay in the section
below each answer on purpose: a maintainer who engages later can overturn a decision by
pointing at a preserved alternative, and the cost is rewriting one requirement rather than
reconstructing the argument.

## 1. Where does the declaration live?

Three options were named in the thread
([lsiepel, 1481931283](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931283)),
plus one from the sketch:

| Option | Sketch | Evidence |
|---|---|---|
| a. Item metadata namespace (`energy:`) | Kai's 2023 proposal | Implemented and field-proven in emsmanager; zero new concepts for users |
| b. "Power description" providers (analogous to state descriptions) | Kai's alternative ([1481931263](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931263)) | Lets bindings ship profiles programmatically; composes with (a) |
| c. Thing/channel annotation by add-ons | lsiepel | Rejected for "logical devices" without Things ([1481931263](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931263)) |
| d. New add-on types for EMS extensibility | Kai 2026 ([5016907260](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5016907260)) | "we will have to work out to what extent this makes sense" |

Working hypothesis worth testing: (a) for user declaration, (b) as the SPI so add-ons
can contribute the same information, (d) for the pluggable pieces (price sources,
forecast sources, algorithms) — mirroring how persistence/transformation services are
add-on types today. Flat metadata demonstrably struggles with wave-3 _named profiles
carrying TimeSeries payloads_ — one reason not to hard-commit to (a) alone.

**ANSWERED — the owner, 2026-08-02 (D5, `docs/OWNER_DECISIONS.md`).** The working
hypothesis, confirmed and tightened: **(a) and (b) both, behind one provider interface** —
`energy:` item metadata for users, a programmatic contribution path for add-ons and
scripts, with nothing above that interface able to tell which mechanism produced a
participant. This is the flagship open question of Kai's own comment, and the answer keeps
his "no Thing required" path while giving bindings somewhere to put a profile they know
about. Core precedent: `MetadataKey` is already `(namespace, itemName)`, so a contributor
can address a user-declared participant with no coordination at all.

_Not chosen, preserved:_ (a) **metadata only** — zero new concepts, and it is what the
reference implementation ships; no add-on can then contribute, and wave-3 profile payloads
have nowhere to live. (b) **a programmatic SPI only** — the cleanest type model; the
user-facing "declare an Item and you are done" path disappears with it. (c) **Thing/channel
annotation by add-ons** (lsiepel) — it puts the declaration where the device already is;
rejected in-thread for logical devices that have no Thing. (d) **a new add-on type for the
declaration plane** — still live for the _pluggable_ pieces (price sources, forecast
sources, algorithms) and untouched by this answer, which is about how a participant is
declared.

## 2. Core vs. add-on boundary

Kai 2026: the final solution belongs in openhab-core, not "yet another binding"
([5016907260](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5016907260)).
The reference implementation's internal seams suggest a concrete split to discuss:

- **Core:** participant model + registry, profile classes, energy levels, the engine
  contract (and the "direct communication in the background" lsiepel asked for in
  [1492968827](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1492968827)).
- **Add-ons:** price providers, forecast providers, regional tariff/constraint logic,
  optimization algorithms (masipila's "contributed algorithms",
  [1481931363](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931363)).

## 3. Engine simplicity

The sketch wanted the `EnergyManagementService` so simple it "could possibly even be a
rule template" ([1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374)) —
scripts must be able to implement alternative algorithms. Whatever lands must keep the
engine swappable rather than monolithic.

## 4. Validation methodology (from the reference)

Before any engine variant steers a live building: unit tests on the pure logic, then a
shadow mode running beside the user's existing automation comparing every decision, then
live control behind a master kill switch. This proved out in production
([5014707411](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5014707411))
and should be a first-class acceptance path in the implementation plan, not an
afterthought.

## 5. Questions raised by the wave-1 prototype

These come from building `org.openhab.core.energy` against this change — **not** from the
issue thread, so a reviewer should weigh them as implementer evidence, not as consensus.
Ids are the prototype's (see `docs/PROTOTYPE_FEEDBACK.md`). Each entry says what is
undecided, which options are on the table, and what the prototype had to do to compile.
What the prototype did is evidence of the cost of the silence, never a recommendation.

### 5.1 How is a participant identity formed, and what is a duplicate? (A12 / B8)

_Participant identity_ now requires one; nothing says how it is formed. Options: supplied
by the user in the declaration; derived from the declaring Item's name; generated by the
registry; or a per-mechanism scheme plus a stated mapping between them. Equally open: what
a second declaration of an identity already present means — a further statement about one
participant, a conflict to surface, or a second participant.

Prototype: the id is an opaque non-blank string, derived from the item name with an
optional override, and uniqueness was left to a registry the prototype does not contain.
Nothing tells a binding which id the user's metadata used, so two mechanisms agreeing on
an id is a coincidence there, not a contract — which is exactly what
`participant-discovery`'s "explicit declaration wins" needs in order to work.

**ANSWERED — the owner, 2026-08-02 (D5, `docs/OWNER_DECISIONS.md`).** Identity is **the
Item name**, overridable by an explicit id in the declaration; a **duplicate is a further
statement about the same participant**, never a second participant and never an error; and
precedence, stated once and used everywhere, is **explicit metadata > contributed (ties by
descending `service.ranking`) > discovered proposal**. One deliberate exception: the
**write-side actuation sink is chosen by explicit configuration naming it, never by
ranking**, with a per-participant override allowed. Identity and the duplicate rule are
this change's, in _Participant identity_; the precedence chain and the sink exception are
normative **once**, in `define-extension-points` (_Deterministic resolution between
contributors_ and _The actuation sink is named, never ranked_), because a collision between
declaration mechanisms is the contribution plane's business — this section keeps the
alternatives for both halves, since they were one decision. The item name is what makes the
coincidence a contract — a binding can address a user-declared participant without any
coordination, because both sides already know the Item. Core precedent for the read-side
aggregation: `StateDescriptionServiceImpl`; for naming a writer: `defaultPersistence`.
Production precedent: the reference's `MetadataParticipantScanner` has merged whiteboard
participants with tagged Items, metadata winning on duplicates, for months.

_Not chosen, preserved:_ (a) **a registry-generated id** — guarantees uniqueness and never
collides; it makes the two-mechanisms-one-device case unimplementable, because neither
mechanism can predict the id. (b) **a user-supplied id, mandatory** — explicit and
rename-proof; it puts a bookkeeping burden on every declaration and has nothing to fall
back on when it is omitted. (c) **a per-mechanism scheme plus a stated mapping** — honest
about the mechanisms differing; the mapping is then a second thing to specify and to keep
correct. (d) **a duplicate is a conflict to surface** — safest-sounding, and it turns "the
user overrides the add-on's guess" into an error the user must hand-resolve. (e) **a
duplicate is a second participant** — needs no rule at all, and the device is then steered
twice. (f) **a ranked actuation sink**, like the read side — one rule for everything; it
lets a site's writer be decided by a binding race, which is why the exception exists.

Consequence for the Item name as identity, worth stating because it is the cost of the
choice: renaming an Item renames the participant. Engine-held state keyed on the identity
therefore does not follow a rename, and a site that needs rename stability declares an
explicit id.

**FOLLOW-UP — the owner, 2026-08-03 (D26, `docs/OWNER_DECISIONS.md`).** One case of "a
duplicate is a further statement" needed an exception, and §5.18 carries it: when the
**higher-ranked** statement is malformed, the chain does not fall through to the next one.
The participant is blocked rather than resolved, so precedence never turns a typo into a
change of owner. Everything else about D5's chain is unchanged, including that a duplicate
is never an error.

### 5.2 Priority — direction, default and tie-break (A4 / E1 / B7)

_Priority_ now requires all three to be stated; the values stay open. Is the better
priority the lower number or the higher one? What does a consumer that declares none get?
How are equal priorities broken, given that whatever is chosen must not reduce to
incidental enumeration order? A fourth, cross-change part: `energy-levels` L1 asks
whether this scale and the level scale should point the same way; today they point
opposite ways.

The tie-break question is broader than a rule over the cycle's inputs, and the requirement
is worded so that it stays that way: a rotation or fair-share between equal-priority
consumers depends on what was served last time rather than on this cycle's inputs, and it
is a normal EMS behaviour — arguably the one a user expects "equal priority" to produce.
Open with the rest: may the tie-break be stateful, or must it be a pure function of the
cycle's inputs? Both satisfy "not incidental enumeration order", which is the only property
the requirement fixes.

Evidence of the ambiguity: this requirement's own source note reads "lower runs first,
higher wins on conflict", which points both ways in one sentence, and
`engine-contract`'s _Deterministic conflict resolution_ reuses the word ("the
higher-priority decision is applied") without asserting a direction either. Only
`fixtures/` disambiguates.

Prototype: adopted the fixtures' reading (lower = better), imposed a total order with the
participant id as the tie-break, and defined no default.

**ANSWERED — the owner, 2026-08-02 (D4, `docs/OWNER_DECISIONS.md`).** **Lower number =
better priority, in both senses** — served first when power is scarce _and_ winning when
two decisions conflict. Default **100** for a consumer that declares none. Tie-break:
**participant id ascending, stateless**, so an overload resolves the same way from the same
snapshot. Several algorithms may be active at once and conflicts between them resolve by
this same rule (`define-engine-contract` §15). Deliberately **not** forced to point the same
way as the level scale: priority is a rank (lower is better, like every rank), a level is a
magnitude (higher means more available), and each is now stated in prose where it is
defined — which is what `energy-levels` L1 asked for.

**Known follow-on, recorded rather than resolved:** the reference binding's live
`Controller` contract says the opposite for the conflict half — "lower runs first, higher
wins on conflict" — and `EmsManagerBridgeHandler` implements it by iteration order.
Adopting lower-wins-conflict is therefore a **departure from running production
behaviour**, and the binding needs reconciling. The earlier claim that
`buildEngineDispatchSet` proved lower-wins was wrong and has been struck: that method reads
no priority number at all.

_Not chosen, preserved:_ (a) **higher number = better** — the direction a core reviewer may
expect from OSGi's `service.ranking`; it inverts every fixture in the corpus, and note that
`service.ranking` governs which _provider_ is used (§5.1) rather than serving order, so the
two conventions do not actually collide. (b) **split directions** — lower runs first,
higher wins on conflict, exactly as the reference binding says today; it is the only option
that requires no binding change, and it is a sentence that points both ways. (c) **a
stateful tie-break** — rotation or fair-share between equal priorities, which is arguably
what a user expects "equal priority" to produce; it makes overload resolution
unreproducible from a snapshot and kills fixture conformance, and it belongs to an
algorithm that owns a priority rather than to the ordering primitive. (d) **no default at
all**, leaving an undeclared consumer implementation-placed — which the requirement itself
forbids.

### 5.3 Is a declared deadline absolute, recurring, or both? (A2)

_Demand declaration_ now requires a deadline to resolve to exactly one instant per
evaluation; which forms may be declared is untouched. "4 kWh ready by 07:00" reads as a
recurring daily local time; "by noon today" reads as an absolute instant. Options:
absolute only; recurring local time only; both; both plus a stated rule for what a bare
local time means. Whichever is chosen also decides the time zone the resolution uses.

Prototype: implemented both forms, the recurring one resolving to the next occurrence at
or after the evaluation reference in the reference's zone. Two variants existed only
because the requirement's own examples read both ways.

### 5.4 Does demand have kinds beyond energy plus deadline? (A3 / B5)

The requirement defines demand as an energy amount with a deadline, then gives two
examples its own shape cannot express: "charged to 80 % by noon" is a state-of-charge
target, and "run 5 h within the next 12 h" is a runtime inside a window. Options: add
those demand kinds; keep the shape and drop the two examples; or convert them to energy
at declaration time, which needs stated inputs (battery capacity, rated power) and moves
the question rather than answering it.

Prototype: modelled the stated shape only — energy, deadline, and a flag for the
"must not be interrupted" case — so neither example is buildable today.

**ANSWERED — the owner, 2026-08-02 (D17, Part B row PM-5.4, `docs/OWNER_DECISIONS.md`),
as a stated default.** The second option: **keep the shape and drop the two examples.**
Demand carries exactly three terms — **an energy amount, a deadline, and a
must-not-be-interrupted flag** — which is what the prototype could build and what the
reference declares in production (`demandKwh` + `deadlineHour` and nothing else). The two
examples are disposed of rather than deleted quietly: a **state-of-charge target is
converted to energy by whoever declares it**, where the pack capacity is known, and the
requirement now carries a scenario saying so; **runtime-inside-a-window is a wave-3
concern** and is not expressible in wave 1 — an honest gap rather than a silent one.

_Not chosen, preserved:_ (a) **add the demand kinds** — a state-of-charge target and a
runtime-in-a-window become first-class declarations; each needs its own scheduling
semantics (an SoC target needs the capacity and the charge curve; a runtime window needs a
window model the corpus does not have), so wave 1 would carry three shapes where two of
them cannot be planned against. (b) **convert at declaration time inside the framework** —
the user still writes "80 %" and the engine converts; it moves the question rather than
answering it, because the conversion then needs declared inputs (capacity, rated power)
that only the device's own integration knows, and a wrong capacity produces a plausible
plan for the wrong energy.

### 5.5 Is a load curve a property of the demand or of the programme? (A7)

Both readings are in the corpus: _Demand declaration_ says "Batch-class demand
additionally carrying a load curve", while the taxonomy it cites treats the curve as a
property of the programme. The answer decides whether a Batch consumer with no pending
demand still has a curve, and whether two demands on one device may carry different
curves.

Prototype: placed the curve on the Batch profile, so a demand can be declared or omitted
independently of it. `add-named-profiles` (wave 3) carries the same curves as
`TimeSeries` and will inherit whatever is decided here.

### 5.6 Load curves — bound, spacing, sample semantics (A8)

Three sub-questions, none answered anywhere: is a curve's length bounded; are samples
evenly spaced over the runtime or do they carry their own offsets; and is a sample an
instantaneous power or the average over its interval? The third changes the energy that
the same curve represents.

Prototype: samples evenly spaced over the declared runtime, values finite and `>= 0` and
not all zero, no upper bound and no sample-semantics statement — so its energy check had
to assume one reading.

### 5.7 Does the level gate — or at least `never` — belong on every consumer class? (A1 / F13)

_Level-gated operation_ scopes the gate to **Simple** consumers; the storm.house taxonomy
it cites scopes it per consumer regardless of class. As the requirement stands, a Batch
dishwasher or a Controllable wallbox cannot be marked hands-off at all, although "devices
the engine must leave alone" is the setting that most obviously applies to every class.
Options: keep it on Simple; move the whole gate onto every consumer; or keep the level
gate on Simple and lift only `never` to every consumer.

Prototype: followed the requirement literally — the gate lives on the Simple profile and
the other three classes report no gate. (The prototype cites this as "A1, F13"; F13 has no
entry in either of its documents, so it is carried as an alias of A1 — see
`docs/PROTOTYPE_FEEDBACK.md`.)

**ANSWERED — the owner, 2026-08-02 (D13, `docs/OWNER_DECISIONS.md`).** The third option:
**the level gate stays on Simple, and `never` lifts off the profile onto the consumer**, so
all four classes can be marked hands-off. That is a structural change, not a wording one —
`never` is no longer a value of the Simple gate but a property of a consumer, and the spec
now carries a separate _Hands-off consumers_ requirement for it, leaving _Level-gated
operation_ to be about the level gate alone. Both are **engine-owned prohibitions** that a
contributed algorithm cannot set aside (`define-engine-contract` §16). The argument that
decided it is a safety one rather than a modelling one: with `never` on Simple only, the
devices most in need of hands-off — an EV you sometimes drive on your own terms, a
dishwasher — cannot have it, so users fake it by deleting the declaration, which also
deletes the measurement the electrical-limit floor needs. storm.house, which the requirement
cites, scopes the gate per consumer regardless of class.

**How far the flag binds — decided by reconciliation, and the other reading kept.** D13
answers hands-off against **contributed algorithms**: a script may not start a device its
owner marked `never`. It says nothing about the engine's own two unconditional behaviours,
the electrical-limit floor and the safe state on a stale safety input, both of which
dispositioned every load with no carve-out. As the corpus stood, an EV charger marked
hands-off was untouchable under one requirement and deferrable under another, and both were
defensible. It is now stated the same way a running Batch programme is treated: the floor
**books** a hands-off load and refuses with a reason rather than shedding it, and the safe
state leaves it alone.

_Not chosen, preserved:_ **hands-off binds algorithms but yields to the electrical-limit
rung** — the flag is a prohibition on optimization, not a claim on physics, so a fuse may
shed a hands-off load like any other. It keeps `define-engine-contract`'s ladder absolute,
with no exceptions to state, and it is the reading a safety-first reviewer reaches for
first. Its cost is that "the engine leaves this device alone" becomes conditional exactly
when a user would most notice it being wrong, and the user has no way to express the
stronger intent at all. A maintainer who prefers it deletes the exception from the ladder,
from _Electrical limits outrank optimization_, from _Safe state on a degraded safety input_
and the scenario here — four sites, one reading.

_Not chosen, preserved:_ (a) **keep everything on Simple** — no model change at all, and it
matches the requirement as written and the prototype as built; its cost is the safety
regression above. (b) **move the whole gate, level threshold included, onto every consumer**
— the most consistent model, and arguably what storm.house does; its cost is that "run at
level ≥ N" has no obvious meaning for a Batch programme with a deadline or for a
ModeControllable device that has its own mode-per-level mapping, so three of the four
classes would carry a parameter nobody can define crisply. That is the option to reach for
if the split ever reads as arbitrary.

### 5.8 Which provider roles may be controllable, and is the clamp mandatory? (A5)

_Controllable providers_ calls a battery "the primary case" and says nothing about a
curtailable PV inverter, which is a real device class, nor about whether a controllable
provider must declare a `[min, max]` clamp at all.

Prototype: allowed any role to be controllable and made the clamp optional. Both choices
are unbacked by the corpus; the opposite of either would also have compiled.

**ANSWERED IN PART — the owner, 2026-08-02 (D16 / pack A10, `docs/OWNER_DECISIONS.md`).**
The clamp half: **`[min, max]` is mandatory for any controllable provider**, and a
controllable declaration without one is a malformed declaration — the participant is
skipped whole and reported (§5.18), not accepted with an unbounded setpoint. Production
precedent: the reference's battery provider carries `min`/`max` in the declaration and its
handler refuses to write without them.

_Not chosen, preserved:_ **an optional clamp** — it lets a device that genuinely has no
usable bound (or whose bound is discovered at runtime) be declared at all; its cost is that
the first unbounded controllable declaration is an unbounded setpoint write to an inverter.

**A second half this section did not ask about, added by reconciliation — the provider's
priority.** D10 defines surplus as grid export **plus the battery charging the engine can
reclaim**, and both requirements that consume it (`define-energy-levels` _Surplus escalation
of the current level_, `define-optimization-objectives` _Selectable objective_) decide
reclaimability by comparing a consumer's priority against **the battery's**. _Priority_
grants a priority to consumers only, so as the corpus stood the comparison had no second
operand and "reclaimable" was undecidable from the cycle's inputs. The requirement now gives
a controllable provider a priority on the same scale with the same default (lower is better,
100), which is the smallest change that makes D10 computable and adds no new concept.

_Not chosen, preserved:_ (a) **a fixed rule instead of a number** — for instance "battery
charging is always reclaimable", or "never below a declared state-of-charge reserve"; it
needs no declaration at all and reads simply, and it takes the choice away from the one
person who knows whether this house charges the battery for the evening or for the week.
(b) **the two-quantity split** (`define-engine-contract` design.md §11): `surplus` stays
grid export only and a second `allocatableBudget` adds the draw of participants the engine
is steering, with the battery served by priority as an ordinary participant of the
allocator — the same answer reached through a different model, and the one to adopt if the
recursion bullet in that section turns out to need answering too.

**Still open:** which provider roles may be controllable — a curtailable PV inverter is a
real device class and the requirement still calls a battery "the primary case" without
saying whether anything else qualifies. Not put to the owner, not answered here.

### 5.9 May a controllable bound be a current as well as a power? (A9)

_Four consumer profile classes_ calls Controllable a continuous **power** profile while its
own scenario is "min 6 A and max 32 A", and _Controllable providers_ specifies clamps in
power — one concept in two dimensions. Options: mandate power everywhere and reword the
wallbox scenario; allow power or current on consumers and leave the conversion to whatever
writes to the device; or allow either on both sides.

Prototype: provider clamps in power, consumer bounds in power **or** current. That
asymmetry is the corpus's, not a design choice. Related: the prototype's B4, filed against
the declaration surface — whether a declared physical quantity carries its unit at all.

**ANSWERED — the owner, 2026-08-02 (D16 / pack A10, `docs/OWNER_DECISIONS.md`).** The
third option, symmetrically: **either dimension is accepted on consumers and on
controllable providers**, the engine converting a current bound to power with the declared
nominal voltage and the participant's phase count, and holding **watts as the canonical
internal unit**. The corpus's asymmetry is removed rather than blessed, and the wallbox
scenario keeps saying 6 A — which is the IEC 61851 minimum single-phase charging current,
so restating it in watts would make the requirement lie about the device. B4 resolves with
it, along the layering core itself uses: **declared configuration values are bare numbers
whose dimension and unit come from the schema** (`maxCurrentA`, `ratedW`, `demandKwh`),
while **item states are `QuantityType` and UoM does the conversion** —
`ConfigDescriptionParameter` carries `unit` for numeric parameters and throws if you set
one on a text or boolean parameter, which is core stating the same split.

_Not chosen, preserved:_ (a) **watts everywhere**, rewording the wallbox scenario — one
dimension, no conversion, no nominal-voltage input to get wrong; every wallbox user then
hand-converts 6 A for their phase count, and gets it wrong on a single-phase site. (b)
**power on providers, current allowed on consumers** — the corpus's current asymmetry made
official; it needs a reason nobody has, since a curtailable inverter is as amperage-natural
as a charger. (c) **units carried in the value on every declaration**, rather than by the
schema — self-describing and immune to a renamed key; it fights core's own config model,
where a config value is a bare number.

### 5.10 How does a four-level site signal map onto _n_ ordered modes? (A6)

The ModeControllable class puts no bound on the number of modes, while the requirement's
SG-ready scenario demands the level-to-mode mapping work "without translation logic in
user rules". No mapping is specified for any _n_ other than four. Options: require exactly
four modes for level-driven steering; define a mapping for any _n_; or make the mapping
declarable per consumer.

Prototype: not modelled — left as engine policy, which means this requirement is presently
unimplemented rather than implemented differently. Related: `energy-levels` L2 (the
SG-ready offset).

**ANSWERED — the owner, 2026-08-02 (D17, Part B row PM-5.10, `docs/OWNER_DECISIONS.md`),
as a stated default.** The second option: **define a mapping for any _n_** —
`modeIndex = ceil(level × (n − 1) / 3)`, zero-based, index 0 the most restricted mode —
with **_n_ = 2 excluded** and steered by the per-consumer "on at level ≥ N" gate instead.
It is now the _Level-to-mode mapping for any mode count_ requirement. Three things worth
holding on to, because each is a place a reader can go wrong:

- **Why `ceil` rather than `round` or `floor`.** The rounding decides what a **normal**
  level does to a device with few modes. With `floor`, normal drops a three-mode device
  into its most restricted mode, which reads as blocking a device the level did not block.
  `ceil` is the production form — `EnergyManagementService.modeIndex` computes exactly this
  expression on a live building.
- **Why _n_ = 2 is excluded.** The formula would place a binary device in its permissive
  mode at every level above blocked, which is a threshold nobody declared. The corpus
  already has the right control for a two-state device, and it is the user's: the level
  gate. (`ceil` is also what keeps the formula from blocking a two-mode device at normal —
  that is evidence for the rounding, not a reason to apply the formula to _n_ = 2.)
- **An index is not a wire value.** `modeIndex` is a position in the consumer's own ordered
  list. Turning index 3 into the number an SG-ready device expects on the wire is the
  **named correspondence** of `define-energy-levels` _SG-ready mode mapping_, never
  arithmetic on the index or on the level code. For _n_ = 4 the index happens to equal the
  level code, which is exactly the coincidence that makes the named mapping worth stating.

**An edge this decision leaves, recorded rather than closed.** _Level-gated operation_
scopes the "on at level ≥ N" gate to **Simple** consumers (D13 kept it there and lifted
only `never` onto every class), while this row hands the two-mode case to that same gate.
Two readings, neither of them decided here: a two-mode ModeControllable consumer may
declare the gate, widening its scope by exactly one case; or a device with two states is
declared **Simple** in the first place and the question does not arise. The second is what
the class definitions suggest — two ordered modes is an ON/OFF device wearing a mode
model — and it is not stated anywhere, so a maintainer closing §5.7 or this row should
close both at once.

_Not chosen, preserved:_ (a) **require exactly four modes for level-driven steering** — no
formula at all, and every level-driven device then speaks the same four-valued language;
its cost is that three-mode and five-mode devices are excluded from level steering
outright, and the class deliberately puts no bound on the mode count. (b) **a per-consumer
declarable mapping** (level → mode, written out by the user) — maximally expressive, and it
lets an odd device be described exactly; its cost is a table on every declaration for a
result the formula gets right in the ordinary case, and the requirement's own promise of
"no translation logic in user rules" starts to read as translation logic in user
configuration. That is the option to reach for the first time a device's modes are not
monotone in availability.

### 5.11 What is the sign convention for each role and for setpoints? (E6)

_Provider roles_ now demands one per role; the conventions themselves are open. Is a
positive battery power reading charging or discharging? Is a positive setpoint on a
controllable provider a command to supply or to absorb? Both are needed for site-load
math, and only the grid role is fixed by the thread.

Prototype: inferred positive = supply/discharge and flagged the inference in its power
estimator, because the arithmetic cannot be written without picking one.

**ANSWERED — the owner, 2026-08-02 (D11, `docs/OWNER_DECISIONS.md`) — and this is the
second decision in the set that overrode the recommendation put to the owner.** One
convention, fixed centrally for every role and for setpoints:

- **grid positive = export** (the thread's own, unchanged);
- **PV positive = producing**;
- **battery positive = charging**;
- **consumers positive = consuming**;
- **a controllable provider's setpoint takes the same sign as its own reading**, so a
  positive battery setpoint commands charging;
- a device that reports the opposite sign is **normalised at the edge** by a declared
  invert flag, never by letting that device carry a convention of its own.

Why it is fixed rather than declared per participant: E6 is the blocker under D10 — surplus
cannot be computed at all until every role's sign is known, and an add-on contributing a
battery reading otherwise guesses, with a failure mode that is a 100 % error looking
perfectly plausible on a chart. Grounded in the reference, which already normalises exactly
this way in one layer (`invertGrid` / `invertSolar` / `invertHouse` / `invertBattery`)
because real inverters disagree, and on the owner's own site, where an unsigned Modbus
register wraps near zero and has to be corrected at the edge for the same reason.

**The override, stated plainly.** The decision pack recommended the opposite battery
direction: **positive = discharge everywhere** — "positive = power flowing in the direction
the role names, toward the site boundary" — under which a battery commanded **−3000 W is
charging**, and which carries a self-checking corollary the chosen convention does not:
`Σ providers − Σ consumers = 0` at the boundary. Its evidence was the reference's own
`SetpointRequest.Kind.WATTS_BATTERY` ("Positive = discharge, negative = charge"), which has
driven a real hybrid inverter for months. The owner chose charging-positive instead. Both
are internally consistent; the corpus states the owner's, and a maintainer who prefers the
pack's flips the battery bullet, the setpoint bullet and the two battery scenarios in
_Provider roles_ — three sentences and no requirement structure. Worth recording alongside
it, because it is the argument for fixing this now rather than later: the reference binding
is **itself inconsistent** on the battery reading today (`Site` and `EnergyReading` say
positive = charging, `EnergyContext` says positive = discharging), and it has never bitten
only because nothing in it computes with the battery reading.

_Not chosen, preserved:_ (a) **the pack's positive = discharge**, above, with its boundary
corollary. (b) **per-participant declared conventions** — every device says which way it
counts and nothing is normalised; it is the most tolerant of hardware, and it makes every
cross-device arithmetic (surplus, site load, allocation) depend on a declaration being
right, with no way to check it. (c) **readings fixed, setpoint direction declared per
device** — honest about inverters whose command channel genuinely disagrees with their
measurement channel; it means a positive command and a positive reading can mean opposite
things on one participant, which is the single most dangerous ambiguity in the set.

### 5.12 Per-phase declaration and per-phase measurement (E2 / E3)

_Phase declaration_ now lets a consumer declare its phases. Three things stay open:

- how a phase is named or identified in a declaration;
- what an omitted phase declaration means — every phase, one unknown phase, or exempt from
  per-phase enforcement;
- whether providers gain a per-phase shape (E3). With a single aggregate power Item per
  provider, per-phase **uncontrolled** load is always empty, so a per-phase limit can only
  constrain what the engine itself dispatched. The alternative is to state outright that
  per-phase enforcement covers controlled load only.

Prototype: carried the phase assignment in engine configuration (`phases=wallbox=1,2,3`)
purely so `engine-contract`'s "Phase-aware headroom" scenario could be tested at all, and
left per-phase uncontrolled load empty.

**ANSWERED — the owner, 2026-08-02 (D16 / pack A10, `docs/OWNER_DECISIONS.md`).** All
three, in one shape: phases are the **integer indices 1, 2 and 3**; **an omitted phase
declaration means exempt from per-phase enforcement**, still constrained by the site total,
and **reported as a declaration gap** wherever the site declares per-phase budgets; and
**providers may declare per-phase readings**, optionally, so a site that wants real
per-phase headroom can have it without invalidating every single-Item provider declaration
in existence. The shortfall the third bullet names is stated rather than papered over:
per-phase enforcement covers **controlled load only** unless a provider declares per-phase
readings (`define-engine-contract` §20). Production precedent: the reference carries
`totalAmpsL1/L2/L3` and computes real per-phase headroom from measured per-phase load.

_Not chosen, preserved:_ (a) **named phases** (L1/L2/L3, or site-defined labels) — friendly
to read and closer to how an electrician talks; it needs a naming authority and a mapping
for every site, and the integers are what a limit calculation uses anyway. (b) **an omitted
declaration means all three phases** — conservative-sounding; it over-counts a single-phase
load threefold and starves the site the first time someone forgets. (c) **an omitted
declaration means one unknown phase** — the worst of both: the engine constrains a phase
the load may not even be on. (d) **mandatory per-phase provider readings** — makes
phase-aware headroom mean what a reader assumes it means; it invalidates every existing
aggregate-reading declaration.

### 5.13 Which power figure does the floor book? (E4)

_Consumer power figure_ now says the model must name the value; which value it names is
open. Options: the Simple class's surplus on-threshold ("typically rated power") doubles
as the rating; a separate rated or maximum power is declared; or the answer is per class —
a Controllable consumer's declared maximum, a Batch programme's curve, a ModeControllable
mode's expected draw (that last one is `engine-contract` E5, which asks whether a mode may
carry a draw at all).

Open with it, and deliberately not settled by the requirement: whether **every** consumer
must have such a value at all, or whether a class may name none and be handled by a stated
policy instead. `define-engine-contract` design.md §10 keeps "mode changes are exempt from
the budget, and the floor catches the consequence a cycle later through measurement" as a live
reading for ModeControllable, and _Four consumer profile classes_ defines that class as
carrying "no power setpoint". A requirement that obliged every consumer to carry a bookable
figure would have closed that reading; this one does not.

Prototype: reused the on-threshold as the rating, which is silently wrong for any user who
sets that threshold with margin — the budget arithmetic then books a number chosen for
switching behaviour.

**ANSWERED — the owner, 2026-08-02 (D15, `docs/OWNER_DECISIONS.md`).** The per-class
answer, and the open half — whether every class must carry a figure — is answered **no**:

- **Controllable** → its declared maximum (its commanded value once commanded);
- **Batch** → rated power scaled by its curve, the curve's peak when admitting it;
- **Simple** → a declared `ratedPower`, **optional**, falling back to the surplus
  on-threshold when absent and **reporting that fallback as a declaration gap, never a
  rejection**;
- **ModeControllable** → **none**; mode changes stay exempt from the planner's budget, with
  an _optional_ per-mode draw that upgrades a mode into it
  (`define-engine-contract` §10).

`ratedPower` is optional precisely so the decision does not invalidate the reference's live
Simple declarations — which, combined with the reject-on-malformed rule (§5.18), would
otherwise break them at parse time. The reading the design note kept alive — "mode changes
are exempt from the budget, and the floor catches the consequence a cycle later through
measurement" — is the one that was adopted, not the one that was closed.

_Not chosen, preserved:_ (a) **the on-threshold doubles as the rating** — no new
declaration, and it is what the prototype did; it silently over-books, always in the same
direction, because a switching threshold is set with margin. (b) **a mandatory rating on
every consumer, every class** — uniform and easy to reason about; it invalidates existing
declarations and forces a fabricated number for SG-ready modes. (c) **a single policy
instead of a per-class rule** — one sentence in the spec rather than four; it cannot
express that a Batch curve and a Controllable maximum are different kinds of number.

**FOLLOW-UP — D29** (owner decision, 2026-08-03; `docs/OWNER_DECISIONS.md`). How the
per-class figure above combines with a live measurement while the participant is running —
D15's "prefer a fresh measurement", which shipped as the larger of the two — is confirmed as
the owner's own reading rather than the implementer's. The argument and both rejected
readings are recorded once, in `define-engine-contract` design.md §9; nothing in this
section changes.

### 5.14 May a consumer declare its own measurement? (N1)

Nothing states whether a consumer may name an Item carrying its live power, nor whether an
engine should prefer such a measurement over an estimate derived from its power figure.
The only corpus hook is `extension-surface`'s _Graceful degradation on contributor loss_
("a live power measurement that feeds the electrical-limit floor"), which is site-level and
never per consumer.

Prototype: a per-consumer measurement item name is load-bearing — the surplus algorithm,
the power estimator and the staleness gate all read it — and no requirement declares it.
Related: `engine-contract` E14 (whether the floor books estimates or measurements).

**ANSWERED — the owner, 2026-08-02 (D15, `docs/OWNER_DECISIONS.md`).** Yes to both halves:
a participant **may declare an Item carrying its live power**, and the engine **prefers a
fresh measurement over an estimate everywhere except when admitting a load that is not yet
running** — where no measurement can say anything, since the load draws nothing. It is now
the _Declared power measurement_ requirement, and it is what makes "a command is an
envelope, not an order" verifiable at all. Production precedent: `measure` exists on any
consumer in the reference today, whose taxonomy states the principle outright — "running
detection and delivered-energy metering trust measurement over commanded state".

**One thing the answer had to be narrowed to say, recorded because the narrowing is a
choice.** Pack A6 carries two sentences that agree only when the measurement is the larger
number: "prefer the measurement wherever both are available" and, on the engine's side, the
floor books `max(declared, measured)` while a participant runs. They part company for a
running load measuring **below** its declaration — an 11 kW wallbox drawing 3.7 kW because
the car is limiting it. Under the first, the floor books 3.7 kW and admits another load
against the difference; under the second it books 11 kW and does not. The requirement now
states the second, and says so by naming `define-engine-contract` _What the limit floor
books_ rather than by asserting a general preference the floor then contradicts. The
measurement still wins wherever it is the fresher answer to the question actually being
asked — running-detection, delivered energy, an overshooting device.

_Not chosen, preserved:_ (a) **no per-consumer measurement** — site-level readings only, as
the corpus stood; it leaves the surplus recursion with nothing but hand-tuned hysteresis and
makes an overshooting device invisible. (b) **measurement mandatory** — the floor then
always books the truth; it excludes every device that has no meter, which is most Simple
loads. (c) **prefer the estimate always**, with the measurement as diagnostics only —
predictable and stateless; it books a number the device demonstrably ignores. (d) **prefer
the measurement unconditionally once running**, which is pack A6's own first sentence and
the reading the requirement was narrowed away from — it is the more truthful picture of the
present moment and it lets a site pack more load under a fixed budget; its cost is that a
device throttled by something outside the engine's control (a car, a thermostat) hands the
floor back headroom it does not really own, and the load it admits against that headroom is
still running when the throttle lifts. A maintainer who prefers it changes this requirement
and `define-engine-contract` _What the limit floor books_ together, or the pair goes back to
disagreeing.

### 5.15 What is a "forced restart", and how is one recognised? (N2 — for @mstormi)

_Simple-consumer protection parameters_ carries "the minimum runtime doubling as the
catch-up time after a forced restart", carried over from storm.house on mstormi's request
([5012316336](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5012316336)).
Nothing defines the event: an external actor switching the Item, a power cut, an openHAB
restart, or the engine re-enabling after a master stop. Options: define it observably;
delete the clause; or keep it as guidance and not as a requirement.

Prototype: could not implement the clause at all — minimum runtime is enforced only from
the last observed state change. Related: `engine-contract` N3, which asks what the source
of elapsed time is and therefore whether protections survive a restart.

**ANSWERED — the owner, 2026-08-02 (D7 refinement, `docs/OWNER_DECISIONS.md`).** The first
option: **define it observably**. A forced restart is **any OFF→ON transition the engine
did not command**, and the minimum runtime applies from that observed transition exactly as
it would from an engine-initiated start. The engine never has to recognise _why_ the device
started, which is what made the clause unimplementable: an external actor, a power cut, an
openHAB restart and a resume after a master stop are indistinguishable at the item level, so
any definition that names those events cannot be built. A definition that names the
transition is free. It rests on the same mechanism as N3 — the Item's own last state change
(`define-engine-contract` §4) — so mstormi's clause and the protection source are one
answer, not two.

_Not chosen, preserved:_ (a) **delete the clause** — it is the only part of the storm.house
protection set with no operational meaning, and deleting it would be honest; it drops a
behaviour mstormi asked for by name. (b) **keep it as guidance, not as a requirement** —
non-binding, so no implementation is wrong; it leaves a safety-adjacent behaviour to
folklore. (c) **enumerate the events** (external switch, power cut, openHAB restart, resume
after stop) — closest to what "forced restart" says in plain English; unimplementable at the
item level, which is the finding that closed it.

### 5.16 Naming — a second collision (A10 — for Kai)

Kai already questioned `EnergyConsumer` in favour of `DemandDescription`
([1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374)),
parked in `define-extension-points/design.md` §2. The prototype hit a second one, and it
sits entirely inside Kai's own two comments: `EnergyProvider` is the participant
sub-interface of `EnergyManagementParticipant`
([1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374))
and, one comment apart, the thing a price binding brings —
"many bindings that each bring their own `EnergyProvider`"
([1483930195](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1483930195)),
the data-contributor role on the extension surface. This change is deliberately
mechanism-neutral and names no types, so the collision bites at the naming pass, not here.
No rename is proposed — that pass should take both at once, and that §2 is the other half
of this note.

**PARTLY ANSWERED — the owner, 2026-08-02 (D17, Part B row PM-5.16 / EP-2,
`docs/OWNER_DECISIONS.md`), as a stated default.** The _shape_ of the pass is settled even
though the word is not: **park `EnergyProvider` for a single combined naming pass** rather
than renaming piecemeal, and when that pass runs, the **participant side keeps
`EnergyConsumer`**, the **provider role is renamed**, and **data contributors take per-kind
names** (`PriceSource`, `ForecastSource`) — which is what this corpus's wave-2 and wave-3
changes already call them. The other half of the note is
`define-extension-points` design.md §2, which carries the same decision from the
contribution plane's side.

**NOT DECIDED, recorded (the owner, 2026-08-02, `docs/OWNER_DECISIONS.md`, Part D
remainder).** That the participant role should not be called `EnergyProvider` is grounded —
`org.openhab.core.common.registry.Provider` already means "contributor of registry
elements" and the EMS SPI will extend it. **Which word replaces it** is taste, and Kai
coined both of the colliding names, so the owner's position is that it is worth one line in
the thread rather than a unilateral rename. This section stays open on purpose.

### 5.17 What makes a reading stale, and does a participant get a say? (E12)

_Filed by the wave-1 prototype against `define-extension-points` design.md §5.7 and
answered here, because the declaration half is this change's._ `extension-surface`'s
_Graceful degradation on contributor loss_ turns on a safety input having "gone stale", and
nothing said what that means: no age, no source of one, and no way for a participant whose
reading updates every 30 seconds to be told apart from one that updates hourly.

**ANSWERED — the owner, 2026-08-02 (D14, `docs/OWNER_DECISIONS.md`).** Staleness is **an
optional per-participant maximum age, plus unreadable / `UNDEF` / `NULL`, which always
trips** whether or not an age is declared. No ageing mechanism is invented for the EMS: a
site that has a source which freezes without ever going undefined uses core's own `expire`
namespace to turn the frozen Item into `UNDEF`, and the same rule then covers it. The
runtime consequence — freeze and floor — is `define-engine-contract` §21.

_Not chosen, preserved:_ (a) **a single site-wide maximum age** — one number to configure,
nothing per participant; it is wrong for any site mixing a 1-second inverter feed with a
15-minute meter. (b) **an EMS-owned ageing or heartbeat mechanism** — independent of how
the site configures its items, and it would catch a frozen reading with no extra setup; it
duplicates `expire` and gives openHAB a second, competing notion of "this reading is old".
(c) **staleness inferred from the reading's own history** (its observed update interval) —
no declaration at all; it cannot distinguish a device that legitimately went quiet from one
that died, which is precisely the safety case.

### 5.18 What happens to a declaration the framework cannot read? (B11 / B12)

_Filed by the wave-1 prototype against `define-extension-points` design.md §5.5 and §5.6,
and answered here because a declaration in error is a participant-model event._ Nothing
said whether a half-readable declaration yields a half-configured participant, and nothing
named a surface on which the problem could appear at all.

**ANSWERED — the owner, 2026-08-02 (D16 / pack A8, `docs/OWNER_DECISIONS.md`).** A
malformed declaration **skips the whole participant — never a partial accept** — reported
through a `ConfigStatusProvider` for the `energy:` namespace, keyed on the Item that
carries it; and an **absent or unrecognised profile class is a configuration error, never
defaulted to Simple**. Core precedent rather than invention: `ConfigStatusProvider`,
`ConfigStatusMessage{INFORMATION,WARNING,ERROR}` and `ConfigStatusInfoEvent` already exist,
are i18n'd, are per-parameter and are consumed by UIs — an EMS-specific error surface would
be the reviewable defect. The complement is Part C's answer for _unknown_ keys, which are
accepted, ignored and logged once: an unreadable declaration is an error, an unrecognised
extra key is not.

_Not chosen, preserved:_ (a) **partial accept** — the device is at least managed, and a
site with one bad key keeps working; it steers a device on half an intent, which for a
protection parameter is a safety failure. (b) **default an unknown class to Simple** —
maximally forgiving, and Simple is the least capable class so it reads as the safe
fallback; it lets the engine switch a Batch programme mid-run, which the taxonomy forbids.
(c) **reject the bad key only, keep the rest** — the middle road, and the one a
configuration parser usually takes; it makes the resulting participant's behaviour depend
on which key failed.

**FOLLOW-UP — the owner, 2026-08-03 (D26, `docs/OWNER_DECISIONS.md`): what the skip does to
the precedence chain.** The wave-1 slice reported an interaction neither D5 nor D16·A8
contemplated. D5 says a duplicate declaration is "a further statement about the same
participant, never an error", with explicit metadata outranking a contributed declaration;
A8 says a malformed declaration "skips the WHOLE participant". Put together: if the user's
own metadata for a device is malformed it is withdrawn, and the add-on's contributed
declaration — until that moment outranked and inert — silently becomes effective. The device
stays managed, on somebody else's terms, from one keystroke. The configuration-status error
makes the typo visible; it does not make the transfer visible, because nothing about the
device's behaviour says which declaration is in force.

**A malformed declaration blocks the participant entirely.** No lower-ranked statement about
that identity takes its place; the device is unmanaged until the declaration is fixed, and
the block lifts by itself when it is. The principle is that **intent to control survives a
wrong text**: a user who wrote a declaration meant to be the one in force, and the failure
mode of honouring that is "nothing happens", which is diagnosable. The failure mode of the
alternative is "something else happens", which is not — the device keeps moving and the
configuration the user is reading is not the one steering it.

_Not chosen, preserved:_ (a) **let the contributed declaration take over** — what the slice
does today, and the reading that keeps a device managed through a configuration error, which
is a real virtue on a site whose add-on declaration is perfectly good; its cost is the
silent transfer of control described above, and it is worst exactly where it matters most,
on a device whose owner cared enough to override the add-on in the first place. (b) **take
over after acknowledgement** — the contributed declaration becomes effective once the user
acknowledges the error on the configuration-status surface; it serves both cases honestly
and is the option to reach for if unmanaged devices turn out to be the bigger complaint; its
cost is a modal decision on the safety path and an acknowledgement mechanism the corpus does
not have and would have to specify — including what an unacknowledged error means across a
restart.
