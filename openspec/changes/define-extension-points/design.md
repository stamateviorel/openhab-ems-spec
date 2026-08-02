# Design notes — the mechanism question (deliberately open)

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

## 3. Are actuation adapters an extension point?

Device-quirk handling on the _write_ side (ACK windows, mode mapping — see
`define-engine-contract` §3) could itself be contributable, e.g. an EEBus add-on acting
as "one implementation behind the interfaces"
([2016826350](https://github.com/openhab/openhab-core/issues/3478#issuecomment-2016826350)).
Open.

## 4. Selection/composition UX

"User selects which contributions are used" needs a home: bridge-style configuration,
per-participant metadata, or MainUI flow. Ties to the discovery change
(`discover-participants-from-model`) for the propose-and-confirm surface.

## 5. What the wave-1 prototype surfaced

Everything in this section came out of building the wave-1 slice against this corpus, not
out of #3478. The identifiers are the prototype's own (see `docs/PROTOTYPE_FEEDBACK.md`).
Each entry is a question this change still owes an answer to; none of them is answered
here.

### 5.1 Is the `energy:` schema core's to specify? (extends §1 — B1)

`energy:` is named in Kai's sketch and cited by `energy-participants` _Declaring
participants on existing Items_, but no requirement anywhere in the corpus defines a
single key, a value grammar or a type for it. Building a parser therefore meant inventing
every key name, and nothing in the corpus can say whether any of them is right.

**For @kaikreuzer:** does core specify the `energy:` metadata schema normatively, or is
that schema explicitly out of core's scope? The answer is downstream of §1 — a metadata
schema is only core's business if metadata is a mechanism core owns.

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

### 5.3 Declarations core cannot act on yet (B3)

A wave-1 mechanism will meet keys belonging to waves 2 and 3 — a provider's `price`, a
consumer's `schedule` — long before anything can act on them. Open: is a declaration
carrying a key the running system cannot act on accepted (and the key ignored until the
capability ships), or rejected? Ties to 5.5, which decides whether such a key is an error
at all.

### 5.4 What an absent or unreadable profile class means (B6)

`energy-participants` _Four consumer profile classes_ says every consumer is one of four;
nothing says what a declaration with no class, or with a class nobody recognises, is. The
two silences are not symmetric — treating an unrecognised class as Simple lets the engine
believe it may switch ON and OFF what is really a Batch programme that must run to
completion. Open: what an absent class means, what an unreadable one means, and whether
they mean the same thing.

### 5.5 What a declaration in error does, and where it shows (B11)

_Runtime contribution_ says contributions are registered when the contributor appears; it
does not say what becomes of one that is malformed. Skipping the participant leaves the
device with whatever automation the user already had, so a typo in a protection parameter
silently un-manages a device; accepting it partially steers the device on incomplete
intent. The corpus has no notion of a "declaration in error" and no surface on which to
show one. Open: skip or partial accept, and where an erroneous declaration becomes
visible.

### 5.6 Where degradation is reported (B12)

_Graceful degradation on contributor loss_ requires that the engine "reports the degraded
source" and never says where — an Item, an event, a REST resource, the log. Open: where
degradation is reported. Note that only the requirement's _scenario_ is data-flavoured;
the requirement text covers any contributor, so a declaration contributor going away is
already inside it.

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

### 5.8 Precedence versus openHAB's own registry contract (B13)

openHAB's `AbstractRegistry` rejects a second element carrying an existing UID — first
come wins, with a warning. That is registration-order dependent, which is the opposite of
what _Deterministic resolution between contributors_ asks for. Open, and it reaches into
openhab-core itself: is the EMS registry not a plain core registry, or does the core
registry contract grow a precedence notion? **For the core registry API maintainers**;
this corpus should not settle it alone.

### 5.9 Script contributors have no unload hook (B14)

_Runtime contribution_'s _Script-contributed consumer_ scenario says the engine treats a
script contribution "exactly like an add-on-contributed one". An add-on's withdrawal is
free — OSGi deactivates it. Nothing in openHAB reliably tells a script that it is being
unloaded, so a reloaded script cannot withdraw its previous contributions the same way.
The two paths have genuinely different lifecycle guarantees, and the scenario asserts they
are equal. Open: does the equivalence claim narrow to something the script path can keep,
or does the corpus require an explicit withdrawal capability of every contributor? Ties to
task 2.1.

### 5.10 Declarations that name Items which never resolve (B15)

`energy-participants` _Declaring participants on existing Items_ is satisfiable without a
Thing, and a mechanism that never consults the `ItemRegistry` is what makes "No Thing
required" trivially true and the mechanism testable on its own. It also pushes a question
onto the engine that no requirement answers: may a declaration name an Item that does not
exist? _Graceful degradation on contributor loss_, in its "Safety input lost" scenario,
covers a measurement that disappears or goes stale, and says
nothing about a declared Item that never resolved in the first place. Open: is an
unresolvable Item name a declaration error (5.5), a runtime condition (5.6/5.7), or both?

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

### 5.12 What "equal terms" means (N10)

_Core-shipped defaults without privilege_ requires core defaults to be replaceable "on
equal terms" and never says what makes terms equal. At least three readings are live: the
same registration mechanism as any contributor, the same priority range, displaceable by
id. They are not equivalent, and each is separately testable, so the requirement is only
as strong as whichever reading an implementer picks. Open: what equal terms means, stated
observably enough that a test can fail.

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
