# Design notes — the mechanism question (answered 2026-08-02)

> **Decision pass, 2026-08-02.** The owner answered this corpus's open questions in a quiz
> session; the record, with rationale and evidence, is
> [`docs/OWNER_DECISIONS.md`](../../../docs/OWNER_DECISIONS.md). Everything marked
> _Answered_ below is an **owner decision in a reference implementation, not thread
> consensus** — one site's ruling, which a maintainer may overturn. Nothing was deleted:
> every option each section carried is left standing, so overturning a decision costs one
> requirement rewrite rather than an archaeology exercise.

## 1. THE open question: what carries a contribution?

Kai 2026: "we might introduce new types of add-ons to make the EMS extensible — but we
will have to work out to what extend this makes sense"
([5016907260](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5016907260)).
Candidate mechanisms, none decided:

| Mechanism | Precedent | Notes |
|---|---|---|
| a. Plain bindings publishing to channels/items | The 2023 sketch's default (price bindings → linked items) | Zero new infrastructure; contribution = items, selection = item configuration |
| b. New EMS add-on types with a dedicated SPI | persistence services / transformations / voice add-ons | First-class lifecycle + typed contracts; the "new types of add-ons" idea |
| c. Scripts / rule templates | automation add-ons, marketplace rule templates | Kai's first-class path for consumers and algorithms; API must be script-friendly |
| d. OSGi service whiteboard | core registries (Item/Thing/Metadata providers) | The runtime register/unregister behaviour in this change matches this precedent |

Likely not either/or: (a) for data-by-items, (d) beneath (b) for typed contributions,
(c) on top via a script-accessible registry. To be worked out with maintainers.

**Answered — D5** (owner decision, 2026-08-02; `docs/OWNER_DECISIONS.md`). This is Kai's
flagship open question, and the answer below is the owner's, not the thread's.

_Decision._ **Both mechanisms, behind one provider SPI.** `energy:` item metadata for
users, an OSGi service whiteboard for add-ons and scripts, unified behind a single
`Provider<EnergyParticipant>`-shaped interface, so nothing above the seam knows which
mechanism produced a participant. **Identity is the Item name**, optionally overridden by
an explicit `id`. A second declaration of an identity already present is **a further
statement about the same participant** — never a second participant, and never an error.
Precedence, stated once and used everywhere: **explicit metadata > contributed (ties by
`service.ranking`) > discovered proposal**. The same rule governs contributed algorithms,
keyed on a `contributorId` the contributor supplies — that is what tells a script reload
from a hijack. One deliberate exception: the **write-side actuation sink is chosen by
explicit configuration naming it, never by ranking** (§3, §5.11).

_Why._ Core's `MetadataKey` is already `(namespace, itemName)`, so a binding can address a
user-declared participant with no coordination; `StateDescriptionServiceImpl` is core's
ranked-aggregator precedent for read-side precedence; and the reference binding's
`MetadataParticipantScanner` has merged whiteboard participants with tagged items,
"metadata winning on duplicate ids", for months.

_Alternatives preserved._ All four mechanism rows above stand: the decision picks up (b)
as the typed contribution surface with (d) beneath it, and keeps (a) and (c) as the
user-facing and script paths over the same SPI. The two losing shapes are recorded in
`docs/DECISIONS_PENDING.md` A1 and remain restorable by narrowing the SPI to one
mechanism — **metadata only** (no add-on can contribute; wave-3 profile payloads have
nowhere to live) and **programmatic SPI only** (the "no Thing required" user-facing path
disappears). Also preserved: treating a duplicate as a conflict, which the decision
rejects because it turns "the user overrides the add-on's guess" into an error the user
must hand-resolve.

_Still open for the thread._ Kai's own framing — to what extent new add-on _types_ make
sense for core — is answered here only for the reference implementation. Task 1.1 stays.

## 2. Naming

Kai himself questioned `EnergyConsumer` — "I am wondering whether something like
`DemandDescription` might be actually a better wording?"
([1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374)).
Parked here so the naming pass happens consciously.

There is a second collision, and the pass should take both at once: `EnergyProvider` is
used in two senses one comment apart, both of them Kai's — the participant sub-interface of
`EnergyManagementParticipant`
([1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374))
and the thing a price binding brings, "many bindings that each bring their own
`EnergyProvider`"
([1483930195](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1483930195)),
which is this extension surface's data-contributor role. The corpus names no types yet, so
this bites at the naming pass rather than today. Surfaced by the wave-1 prototype rather
than by the thread and written up in `define-participant-model` design.md §5.16 (A10, see
`docs/PROTOTYPE_FEEDBACK.md`). No rename is proposed in either place.

**Partly answered — D17** (Part B, PM-5.16/EP-2), **and deliberately left open as a thread
question** (Part D remainder, `docs/OWNER_DECISIONS.md` final row).

_Decided._ Park `EnergyProvider` for a single combined naming pass rather than renaming
piecemeal. When that pass runs: the participant side keeps `EnergyConsumer`; the provider
**role** is renamed; data contributors get **per-kind** names (`PriceSource`,
`ForecastSource`), which is what the corpus's wave-2/3 changes already do.

_Not decided, on purpose._ **Which word replaces `EnergyProvider`.** That the participant
role should not carry that name is grounded — `org.openhab.core.common.registry.Provider`
already means "contributor of registry elements", and under D5 the EMS SPI extends exactly
that — but the replacement word is **taste**, and Kai coined both of the colliding names.
It is worth one line in #3478 rather than a unilateral rename. Task 1.2 stays open and now
carries that shape.

## 3. Are actuation adapters an extension point?

Device-quirk handling on the _write_ side (ACK windows, mode mapping — see
`define-engine-contract` §3) could itself be contributable, e.g. an EEBus add-on acting
as "one implementation behind the interfaces"
([2016826350](https://github.com/openhab/openhab-core/issues/3478#issuecomment-2016826350)).
Open.

**Partly answered — D5.** Whatever an actuation adapter turns out to be, **the site's
write-side sink is chosen by explicit configuration naming it** — never by
`service.ranking`, never by whichever component wins a service race — with a
per-participant override allowed (§5.11). Still open, and left to task 1.3 jointly with
`define-engine-contract`: whether an actuation adapter is a first-class **contribution
kind** with its own SPI (the EEBus "one implementation behind the interfaces" shape) or
device-quirk handling that stays inside the engine. The decision constrains selection, not
contributability.

## 4. Selection/composition UX

"User selects which contributions are used" needs a home: bridge-style configuration,
per-participant metadata, or MainUI flow. Ties to the discovery change
(`discover-participants-from-model`) for the propose-and-confirm surface.

**Answered — D5 and D17** (Part B, EP-4(i)).

_Decision._ Data-contributor selection is **per-role configuration naming the preferred
contributor, defaulting to the highest `service.ranking`** when the site names none —
`StateDescriptionServiceImpl` sorts providers by rank descending, so this is core's own
convention rather than a new one. Participant collisions are not a selection question at
all: they resolve on the one precedence chain from §1 (explicit metadata > contributed >
discovered). The UI home is `define-energy-ui`'s guided-setup surface; the
propose-and-confirm surface stays with `discover-participants-from-model`.

_Exception._ The write-side actuation sink, which is named explicitly and never ranked
(§3, §5.11).

_Alternatives preserved._ Bridge-style configuration and per-participant metadata both
remain viable **homes** for that configuration — the decision fixes the rule, not the
widget.

## 5. What the wave-1 prototype surfaced

Everything in this section came out of building the wave-1 slice against this corpus, not
out of #3478. The identifiers are the prototype's own (see `docs/PROTOTYPE_FEEDBACK.md`).
Each entry was a question this change owed an answer to; the 2026-08-02 decision pass
answered all thirteen, and each answer names its decision id.

### 5.1 Is the `energy:` schema core's to specify? (extends §1 — B1)

`energy:` is named in Kai's sketch and cited by `energy-participants` _Declaring
participants on existing Items_, but no requirement anywhere in the corpus defines a
single key, a value grammar or a type for it. Building a parser therefore meant inventing
every key name, and nothing in the corpus can say whether any of them is right.

**For @kaikreuzer:** does core specify the `energy:` metadata schema normatively, or is
that schema explicitly out of core's scope? The answer is downstream of §1 — a metadata
schema is only core's business if metadata is a mechanism core owns.

**Answered — D18** (Part C, EP-5.1), as a consequence of D5. **Yes, core specifies it**:
whoever owns the framework bundle owns the namespace and publishes it as a
`MetadataConfigDescriptionProvider` — machine-readable, not prose. Core already does this
for `autoupdate` and `expire`, whose javadoc says "every extension which deals with
specific metadata (in its own namespace) is expected to provide an implementation of this
interface". D5 makes metadata a mechanism core owns, so the schema is core's. This is the
highest-leverage answer in the set: it also discharges most of §5.2 and §5.3's surface and
much of `define-energy-ui`'s guided-setup requirement, since UIs render an editor form
from a config description automatically. The question addressed to @kaikreuzer above is
answered here **by the owner**, and is still worth confirming in the thread.

### 5.2 How a declared quantity states its unit (B4)

_Expressive declaration surface_ requires a declared physical quantity to be unambiguous
as to its unit. It deliberately does not say how, and three shapes are in play: keys that
bake the unit into the name (a `…Kwh` / `…Hours` / `…W` suffix); values that carry their
own unit, for which core already has UoM quantity types; and a canonical unit fixed by the
specification per quantity (all power in W, all energy in Wh) with values left as bare
numbers — which removes the ambiguity without changing the declaration surface at all, and
is what openHAB core does widely today. The vocabulary that exists today mixes the first
two, so a bare `3000` has no defined dimension. Open: which shape, or a combination.
Mandating UoM would be a mechanism pick and belongs with §1's answer, not in a
requirement; so, equally, would mandating a canonical unit, which is why the requirement's
scenario says the unit is fixed by a stated rule and does not say the declaration must be
the thing that carries it.

**Answered — D16** (pack A10), with §5.1's mechanism. _Decision:_ **declared config values
are bare numbers whose dimension and unit come from the schema** (`maxCurrentA`, `ratedW`,
`demandKwh`), while **item states are `QuantityType`** and UoM does the conversion. Either
dimension is accepted on a declaration — watts or amps — with **watts canonical**
internally and current converted as declared nominal voltage × phase count. Evidence:
`ConfigDescriptionParameter` carries `unit`/`unitLabel` for INTEGER/DECIMAL and **throws**
if you set them on TEXT/BOOLEAN, which is core saying that config is bare numbers plus a
schema unit while state is `QuantityType`. The normative units requirement belongs to
`define-participant-model` (A10); this change's _Expressive declaration surface_ states the
rule for the declaration surface and defers there. _Alternatives preserved:_ the three
shapes above all stand — the decision picks the third (canonical unit fixed by the
specification) in its schema-carried form, and leaves unit-suffixed key names as the
spelling that survives inside it.

### 5.3 Declarations core cannot act on yet (B3)

A wave-1 mechanism will meet keys belonging to waves 2 and 3 — a provider's `price`, a
consumer's `schedule` — long before anything can act on them. Open: is a declaration
carrying a key the running system cannot act on accepted (and the key ignored until the
capability ships), or rejected? Ties to 5.5, which decides whether such a key is an error
at all.

**Answered — D18** (Part C, EP-5.3/PP-5/B3). **Accept, ignore, log once at DEBUG. Reject
nothing.** `ConfigDescriptionValidatorImpl.validate()` iterates the described parameters
and never the supplied map, so undescribed keys are silently accepted everywhere in
openHAB today. Note this is a different case from a **malformed value**, which skips the
whole participant (§5.5): an unknown key is not an error, a bad value is.

### 5.4 What an absent or unreadable profile class means (B6)

`energy-participants` _Four consumer profile classes_ says every consumer is one of four;
nothing says what a declaration with no class, or with a class nobody recognises, is. The
two silences are not symmetric — treating an unrecognised class as Simple lets the engine
believe it may switch ON and OFF what is really a Batch programme that must run to
completion. Open: what an absent class means, what an unreadable one means, and whether
they mean the same thing.

**Answered — D16** (pack A8). **An absent or unrecognised profile class is a configuration
error, reported, and never defaulted to Simple.** The two silences therefore mean the same
thing: a class that cannot be resolved is an error against the participant, which is
skipped whole (§5.5) and reported. The asymmetry the section names is resolved by refusing
to guess in either direction rather than by picking a safer guess. _Alternative
preserved:_ defaulting an unreadable class to Simple, rejected because it lets the engine
switch a Batch programme mid-run.

### 5.5 What a declaration in error does, and where it shows (B11)

_Runtime contribution_ says contributions are registered when the contributor appears; it
does not say what becomes of one that is malformed. Skipping the participant leaves the
device with whatever automation the user already had, so a typo in a protection parameter
silently un-manages a device; accepting it partially steers the device on incomplete
intent. The corpus has no notion of a "declaration in error" and no surface on which to
show one. Open: skip or partial accept, and where an erroneous declaration becomes
visible.

**Answered — D16** (pack A8). **A malformed declaration skips the WHOLE participant —
never partial-accept — and is reported through a `ConfigStatusProvider` for the `energy:`
namespace, keyed on the item**, alongside the engine's status Item and its deduplicated
decision events (§5.6). Core's `ConfigStatusProvider` /
`ConfigStatusMessage{INFORMATION,WARNING,ERROR}` / `ConfigStatusInfoEvent` already exist,
are i18n'd, are per-parameter and are consumed by UIs — inventing an EMS-specific error
surface would have been the reviewable defect. _Alternatives preserved:_ partial
acceptance (steers a device on incomplete intent) and silent skipping (a typo'd
declaration looks like a working one) both stand above as the losing readings. The cost of
the chosen one is exactly the one the section names — a typo un-manages a device — which
is why the reporting half is not optional.

**FOLLOW-UP — D26** (owner decision, 2026-08-03; `docs/OWNER_DECISIONS.md`). The cost above
turned out to be understated. A malformed declaration does not merely un-manage the device:
where a **contributed** declaration exists for the same identity, withdrawing the
higher-ranked explicit one promotes the contribution, and the device carries on being
steered on somebody else's terms. **A malformed declaration blocks the participant
entirely** — no lower-ranked statement inherits its place — so a typo degrades to "nothing
happens" and never to "something else happens". The rule is normative in
`define-participant-model` _Malformed declarations are reported, never partially accepted_,
and _Deterministic resolution between contributors_ above now says the chain does not
promote past a broken link; alternatives are preserved in
`define-participant-model` design.md §5.18.

### 5.6 Where degradation is reported (B12)

_Graceful degradation on contributor loss_ requires that the engine "reports the degraded
source" and never says where — an Item, an event, a REST resource, the log. Open: where
degradation is reported. Note that only the requirement's _scenario_ is data-flavoured;
the requirement text covers any contributor, so a declaration contributor going away is
already inside it.

**Answered — D16** (pack A8). One observability surface, shared by everything that has to
be visible: **an outcome enum plus a free-text reason on every decision**
(`APPLIED, SHADOWED, STOPPED, SUPERSEDED, DEFERRED, SUPPRESSED, WITHHELD, REJECTED`),
**published as events, deduplicated so an unchanged decision is not re-emitted**, with the
current cycle readable over REST, plus **one engine-published status Item** (String
summary + Number count) that the energy pages read. Core writes no Item for decisions, so
"shadow writes nothing" stays literally true. The normative requirement belongs to
`define-engine-contract`; `define-energy-ui`'s overview minimum set carries the status
line. _Alternative preserved:_ logs only — rejected because it leaves the corpus's own
side-by-side validation scenario untestable.

**FOLLOW-UP — D23** (owner decision, 2026-08-03; `docs/OWNER_DECISIONS.md`). The surface is
unchanged; **who provides each half of it** is now split. The deduplicated events are the
framework's own, because posting an `Event` is not an Item write; the status Item and the
REST view belong to a separate publishing component, because one of them is an Item write
and the other needs a dependency outside the default-library set. A site running the
framework alone therefore still sees a degraded contributor as an event — which is what
_Graceful degradation on contributor loss_ above now says. The three blockers and the two
rejected shapes are recorded once, in `define-engine-contract` design.md §23.

### 5.7 Staleness has no age, and the safe state has no floor (E12)

_Graceful degradation on contributor loss_'s _Safety input lost — degrade safe, not blind_
scenario turns on two undefined terms. "Goes stale" carries no age, so an age-based
staleness test cannot be implemented at all: only an unreadable measurement can trip the
safe state, and a measurement that silently froze hours ago cannot. "A conservative safe
state (floor levels or pause)" names no floor. Open, with the shapes each answer could
take:

- **What staleness is** — a declared maximum age per measurement; unreadable-only, with no
  age notion at all; or both, with the age optional and the unreadable case always tripping.
  An age can be site-wide, per participant, or derived from the source's own observed
  update rate.
- **What the conservative safe state is** — pause every engine-steered load; floor each at
  its declared minimum (which presumes _Consumer power figure_'s answer, and has no meaning
  for a Simple consumer); or freeze at the last dispatched value and refuse any increase.
  The three differ most for a site whose measurement dies mid-ramp.

Flag: until an age exists, the age half of this safety requirement is inert by default in
any faithful implementation.

**Answered — D14** (pack A15). _Staleness_ is **both**: an optional per-participant
maximum age **plus** unreadable / `UNDEF` / `NULL`, which always trips. No new age
mechanism is invented — a site may use core's `expire` namespace to turn a frozen item
into `UNDEF`, which is what makes the age half implementable at all and lifts the flag
above. _Safe state_ is **freeze and floor**: refuse any increase, floor Controllable loads
at their declared minimum, drop ModeControllable to its most-restricted mode, turn Simple
loads off — **all subject to device protections**, so a Simple load inside its minimum
runtime is held rather than shed, and **never interrupting a running Batch programme**.
Grounded in live site behaviour: when the measurement bridge dies, every charger caps at
6 A rather than stopping. The normative statement belongs to `define-engine-contract`;
this change's _Graceful degradation on contributor loss_ scenario now states it and defers
there. _Alternatives preserved:_ pause-everything ("how a user learns to uninstall an
EMS") and keep-running-on-last-known-values (cannot tell a 5-second gap from a 5-hour one)
both stand above.

### 5.8 Precedence versus openHAB's own registry contract (B13)

openHAB's `AbstractRegistry` rejects a second element carrying an existing UID — first
come wins, with a warning. That is registration-order dependent, which is the opposite of
what _Deterministic resolution between contributors_ asks for. Open, and it reaches into
openhab-core itself: is the EMS registry not a plain core registry, or does the core
registry contract grow a precedence notion? **For the core registry API maintainers**;
this corpus should not settle it alone.

**Answered — D18** (Part C, EP-5.8), **and the framing above is withdrawn as stale.** No
core API change is needed and this is not a core-registry-maintainer question: the
participant plane is an **aggregating service over ranked providers**, not an element
registry, so `AbstractRegistry.added()`'s first-come-wins never applies to it.
`StateDescriptionServiceImpl` and `ConfigDescriptionRegistry` are the ranked-aggregation
pattern that exists in core for exactly this. The sentence "it reaches into openhab-core
itself … this corpus should not settle it alone" is left above for the record and is no
longer true.

### 5.9 Script contributors have no unload hook (B14)

_Runtime contribution_'s _Script-contributed consumer_ scenario says the engine treats a
script contribution "exactly like an add-on-contributed one". An add-on's withdrawal is
free — OSGi deactivates it. Nothing in openHAB reliably tells a script that it is being
unloaded, so a reloaded script cannot withdraw its previous contributions the same way.
The two paths have genuinely different lifecycle guarantees, and the scenario asserts they
are equal. Open: does the equivalence claim narrow to something the script path can keep,
or does the corpus require an explicit withdrawal capability of every contributor? Ties to
task 2.1.

**Answered — D18** (Part C, EP-5.9), **and the premise above is false.** Scripts do have
an unload hook: `org.openhab.core.automation.module.script.providersupport`'s
`ProviderScriptExtension` "ensures that they are removed when the script is unloaded", and
`ProviderRegistry.removeAllAddedByScript()` performs the removal. The add-on/script
equivalence in _Runtime contribution_ therefore **stands as written**. D5 adds a second,
independent guarantee: a re-registration under the same `contributorId` is a further
statement about the same participant, never a second one — which is what distinguishes a
script reload from a hijack even if a hook is ever missed. The sentence "Nothing in
openHAB reliably tells a script that it is being unloaded" is left above for the record
and is stale.

### 5.10 Declarations that name Items which never resolve (B15)

`energy-participants` _Declaring participants on existing Items_ is satisfiable without a
Thing, and a mechanism that never consults the `ItemRegistry` is what makes "No Thing
required" trivially true and the mechanism testable on its own. It also pushes a question
onto the engine that no requirement answers: may a declaration name an Item that does not
exist? _Graceful degradation on contributor loss_, in its "Safety input lost" scenario,
covers a measurement that disappears or goes stale, and says
nothing about a declared Item that never resolved in the first place. Open: is an
unresolvable Item name a declaration error (5.5), a runtime condition (5.6/5.7), or both?

**Answered — D17** (Part B, EP-5.10). **A runtime condition, not a declaration error.**
The parser accepts it — item and metadata arrive in either order, and `ItemChannelLink` is
core's precedent that a forward reference is legal and re-binds later — and the engine
treats it at evaluation as a degraded source, logged at **WARNING, not ERROR**.
_Alternative preserved:_ treating it as a declaration error (§5.5), rejected because it
would make declaration order load-bearing for a file-based site. Also preserved: **accept
silently**, with no report at all until the reading is needed — quietest, and it makes a
typo'd Item name indistinguishable from a device that is simply idle. The decision is now
stated as the _A declared Item name that does not resolve is a runtime condition_
requirement, whose third scenario draws the line against `energy-participants` _Malformed
declarations are reported, never partially accepted_: an unreadable declaration is skipped
whole, an unresolved name is not.

### 5.11 Who writes? Choosing the actuation adapter (extends §3 and §4 — N9, B10)

§3 asks whether actuation adapters are contributable; §4 asks how a user selects between
contributors in general. Neither covers the case between them: how a site picks _the_
component that turns decisions into writes. It is the most consequential selection in the
system and the only one where getting it wrong moves hardware. Open as a distinct case,
with candidate shapes: a single adapter named in configuration and used for everything; a
ranked list with fallback when the preferred adapter cannot handle a participant; or
per-participant adapter selection, declared alongside the participant. They differ in what
a site has to state before anything can be written at all. One property is worth stating
whatever the answer: it must not be settled by whichever component happens to win a
service race.

B10 is recorded here as **not a new question**. §4 already says selection/composition needs
a home; the prototype's configuration PID carrying `sources` + `precedence` is evidence for
that question, not an addition to it.

**Answered — D5** (its one deliberate exception). **A single sink named in explicit
configuration, with a per-participant override allowed — never a ranking, never a service
race.** The property the section asked for whatever the answer is now the stated rule
rather than a hope. Direct consequence, stated in the requirement: a site that names no
sink has no sink, so nothing is written and the gap is reported — which is the price of
refusing to let a binding race decide who moves the hardware. _Alternatives preserved:_
the ranked-list-with-fallback and per-participant-only shapes both stand above; the
decision takes the first shape and folds the third in as an override.

### 5.12 What "equal terms" means (N10)

_Core-shipped defaults without privilege_ requires core defaults to be replaceable "on
equal terms" and never says what makes terms equal. At least three readings are live: the
same registration mechanism as any contributor, the same priority range, displaceable by
id. They are not equivalent, and each is separately testable, so the requirement is only
as strong as whichever reading an implementer picks. Open: what equal terms means, stated
observably enough that a test can fail.

**Answered — D18** (Part C, EP-5.12). **The same SPI as any contributor, with no
special-casing, and core's own default registered at `service.ranking = -2`** so any
alternative outranks it with zero configuration — verbatim what
`DefaultStateDescriptionFragmentProvider` does (with `ChannelStateDescriptionProvider` at
`-1`). That is testable, which is what the section asked for. _Readings preserved:_ "the
same priority range" and "displaceable by id" are the two the decision did not take.

### 5.13 What the two new requirements rest on

_Deterministic resolution between contributors_ can only be implemented once "the same
participant" is defined. Participant identity is undefined across the corpus (A12/B8) and
the amendment for it is filed in `define-participant-model`; until it lands, two
mechanisms agreeing on an id is a coincidence rather than a contract. The neighbouring
precedence statement, `participant-discovery` _Explicit declaration wins_, is keyed on
**the same item** rather than on a participant identity — a second reason the two planes
cannot yet be lined up.

Whether an explicit declaration outranks a contributed one is the question the prototype
had to extrapolate an answer to; it belongs to §4 and is deliberately left there.

**Answered — D5.** Participant identity **is the Item name**, optionally overridden by an
explicit `id`, so "the same participant" is defined and _Deterministic resolution between
contributors_ becomes implementable. The two planes line up as one chain — explicit
metadata > contributed (ties by `service.ranking`) > discovered proposal — so
`participant-discovery` _Explicit declaration wins_ and this change's resolution
requirement are **the same rule seen from two sides**, which is exactly what
`discover-participants-from-model` design.md §5 asked. The extrapolation the prototype had
to make (explicit outranks contributed) turns out to be the decided answer; it is now
stated rather than assumed.

## 6. The security boundary: what a contributed algorithm may not do

Not a prototype defect id and not a thread question — the corpus simply never said whether
a contributed algorithm may override a user's "hands off" flag or level gate, and the
wave-1 prototype had to build a seam with a default ("engine-enforced, because it is the
safer reading") and say so in the code. Three readings were live:

| Reading | Consequence |
|---|---|
| Everything is an algorithm input | A contributed script can start a device its owner marked hands-off |
| Everything is engine-owned | No algorithm can lift a gate under deadline pressure, which a real planner legitimately does |
| Split: prohibitions engine-owned, preferences are inputs | Both cases served, along one stated line |

**Answered — D13** (owner decision, 2026-08-02; `docs/OWNER_DECISIONS.md`). Not thread
consensus.

_Decision._ **Prohibitions are engine-owned; preferences are algorithm inputs.** The
engine-owned list is **closed**: the `never` hands-off flag; a user-declared per-consumer
level gate ("run only at level ≥ N"); the readiness interlock; device-protection
constraints (min on/off, max off); and the enumeration of readings that count as **safety
inputs** — a contributor may not claim that status for itself, and a forecast series is by
construction not in it. Overridable, and deliberately so: **the engine's own level-derived
steering**, so a contributed planner can still lift the engine's gate under deadline
pressure.

_Also decided here._ `never` **lifts off the Simple class onto the consumer**, applying to
all four classes. Its requirement lives in `define-participant-model`; its consequence for
named profiles is stated in `add-named-profiles` (`device-profiles` _Hands-off outranks
profile selection_).

_Why._ A user-declared "leave this device alone" that a contributed script can override is
not a setting, it is a suggestion. Grounded in the reference binding, which enforces the
same shape structurally — gating lives in the asset handler below every controller, "the
last line before an item is written" — and in storm.house, which scopes the gate per
consumer regardless of class. Keeping `never` on Simple only would mean the devices most
in need of hands-off (an EV you drive manually, a dishwasher) cannot have it, so users fake
it by deleting the declaration, which also deletes the measurement the limit floor needs —
a safety regression caused by a modelling choice.

_Alternatives preserved._ Both rejected readings stand in the table above and can be
restored by moving one line: making the whole list an algorithm input, or making the
engine's own steering engine-owned too. The cost of the first is the pool-pump use case
("only when it's cheap") becoming advisory; the cost of the second is a planner that
cannot meet a deadline.

### 6.1 A contributor with a genuine protection claim (D25)

**Opened by the wave-1 slice (`org.openhab.core.energy` STAGE1_REPORT.md §3.4).** D13
answers what a contributor may not **invent**. It says nothing about a contributor with a
claim that is simply **true** — a binding that knows its own compressor's duty cycle better
than the metadata does, or the peak-shaver that the slice's own `DecisionKind` JavaDoc names
as the legitimate electrical-limit case. The slice took the strict reading and normalised
every proposal from a non-engine algorithm down to the level-gate rung. That is unforgeable
— the marker interface lives in a `Private-Package` — and it is a real reduction in what a
contribution can be, taken on a reading nobody had made explicitly.

**Answered — D25** (owner decision, 2026-08-03; `docs/OWNER_DECISIONS.md`). Not thread
consensus.

_Decision._ **Corroborate the claim against the declaration.** A contributed decision is
honoured at the device-protection rung when, and only when, the engine can see the same
thing independently, from the same cycle's snapshot. Three conjuncts, all testable, all
stated in _A contributed protection claim is corroborated or demoted_:

1. the participant's own **effective declaration** carries the protection being claimed —
   the declaration that survived precedence, not one the contributor supplied with the
   decision;
1. that protection is **due at this cycle's evaluation**, measured the way every other
   enforcement point measures it (`define-engine-contract` _Protection timing comes from
   device state history_);
1. the **action the decision renders is the action that protection requires** — a maximum
   OFF time that has elapsed requires ON, a minimum OFF time still running requires staying
   off, and a claim pointing the other way is not corroborated by it.

Anything failing one of the three is **demoted to the level-gate rung**, judged there on its
merits, and **reported with the reason**, so a contributor learns why rather than watching
its decision quietly not happen.

_Why these three and not a weaker test._ Each closes a way the claim could otherwise
certify itself. Without (1) a contributor asserts a protection nobody declared; without (2)
it asserts one that is not due, which is every cycle; without (3) a declared, due protection
becomes a general-purpose licence — the fridge's cooldown used to justify switching it on.
And **protection-unknown does not corroborate**: an unreadable history is the absence of
evidence, and reading it as corroboration would put the strongest reachable rung within
reach exactly on the sites least able to check it.

_What this does not open, stated rather than left to be discovered._ The **electrical-limit
rung** has no declaration to corroborate a claim against — a site's limits are the engine's
own inputs, not a participant's — so a contributed decision still cannot reach it, and the
peak-shaver half of the question the slice raised is **not** closed by D25. It stays open
here rather than being read into an answer that did not mention it.

_Alternatives preserved._ (a) **The strict cap the slice shipped** — a contributed decision
never rises above the level gate, full stop. One rule, unforgeable, nothing to test and
nothing to get subtly wrong; its cost is that a binding which genuinely knows its device's
duty cycle, and a peak-shaver of the kind the corpus itself calls legitimate, cannot be
written as a contribution at all. It is the option to return to if corroboration turns out
to be fragile in practice. (b) **Trust the claim as made** — a contributor labels its
decision and the engine honours the label; maximum flexibility, zero mechanism, and it is
what the slice's own record type allowed before D13; its cost is that it re-opens precisely
the escalation D13 closed, since the label is a field anybody can set.
