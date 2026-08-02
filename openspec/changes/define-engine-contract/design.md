# Design notes — open questions

Sections 1–5 predate the wave-1 prototype and now carry its findings underneath them;
sections 6–20 were opened by it outright. Every paragraph that names a defect id
(E\*, N\*, I\*) is prototype-sourced, not thread-sourced — see `docs/PROTOTYPE_FEEDBACK.md`.
All of them are recorded, none is answered. Where the prototype had to behave one way in
order to compile and run, that behaviour is reported as a build artefact, never as a
proposal.

## 1. Evaluation cadence

Kai suggested pull-based scraping "possibly up to every minute", with push "in a future
iteration" ([1482078918](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1482078918)).
The reference runs a short fixed tick plus a debounced re-evaluation on relevant item
changes. Open: fixed tick only, tick + event-responsive, and what the default cadence is.
The requirement stays cadence-neutral ("regular cadence") until decided.

The prototype re-reported this as **E11** after picking 60 s for itself. Nothing changed
here: the question was already recorded, and the 60 s is a build artefact.

## 2. Shadow granularity and the master stop

Is the master stop the same control as global shadow mode (the reference's approach), or
a separate mechanism? And is shadow global-only, or also per-algorithm/per-participant
(the reference supports both layers)? Needs a decision before API design.

**E9 — the two defaults coincide under the unified reading.** _Shadow mode_ defaults to
shadow "on first install"; _Master stop_ describes an engaged stop. If the two are one
control, those are the same statement, and neither requirement acknowledges the overlap.
This is not a separate decision — it is a consequence of this one. Whoever answers
"one control or two" also has to say what a fresh install is in the answer's own terms:
shadowed, stopped, or both. Related: §6, where "stopped" and "shadowed" differ only if the
stop halts more than dispatch.

## 3. Where acknowledgement handling lives

The ACK window could live in the core engine, or in per-device adapters (the reference
puts it in asset handlers so "controllers stay ignorant of item names and write
mechanics"). Related: whether actuation adapters are themselves an extension point (see
`define-extension-points`).

**E8 — what "unacknowledged" means operationally.** The requirement now demands that the
window have a defined length _and_ a defined behaviour on expiry, because without either
one a device that never echoes leaves the engine's behaviour towards it undefined. Four
things it deliberately still does not say — the prototype needed a configuration knob for
each:

1. **How long the window is** — a fixed duration, per-device, per-participant, or derived
   from the device's observed behaviour.
1. **What expiry means** — the outstanding command lapses and actuation for that device
   resumes, or the device is treated as unobservable and withheld from further commands
   until something clears it. The two differ for exactly the device the requirement was
   written for, and the second is the safety-conservative reading: a device whose state
   you cannot observe is quarantined and surfaced as an error rather than commanded blind.
   A maintainer who prefers a third reading — never re-send, never give up, block
   indefinitely — should say so; that is a change to the requirement rather than a
   configuration of it, and this sharpening deliberately does not treat it as already
   available.
1. **What counts as an acknowledgement** — exact equality, a tolerance band, or a
   device-specific comparison. Is 15.999 A an acknowledgement of 16 A? A tolerance is a
   number, and no requirement carries one.
1. **What happens to a changed command** — when a _different_ command for the same device
   is produced while one is outstanding: superseded immediately, queued behind the window,
   or suppressed until the window closes. The three differ in exactly the case that
   matters, a fast-falling budget.

## 4. Context snapshot content

"Same consistent snapshot" needs a defined minimum: live powers, prices, forecasts,
levels, per-participant state. The reference's context object is prior art; the core
version should be decided with the participant-model mechanism (wave-1 design.md §1).

**E17 — may an algorithm retain a snapshot beyond its cycle?** The minimum-content half is
the paragraph above. The other half is lifetime, and it is unrecorded: an algorithm that
keeps a reference to last cycle's snapshot silently defeats the one-snapshot guarantee the
requirement exists to make. Options: retention is forbidden and a snapshot is scoped to the
call it was handed to; a snapshot is an immutable value, so keeping one is harmless as long
as nothing reads it as a view of the present; or retention is explicitly supported, because
an algorithm that compares one cycle against the last is a legitimate thing to write. The
third makes the snapshot part of the public API.

**N3 — where elapsed time for the protections comes from.** `minOn`, `maxOn`, `minOff` and
`maxOff` (`define-participant-model`, _Simple-consumer protection parameters_) are all
measured from the last state change, and no requirement names the source. Candidates, all
behaving differently across an openHAB restart: the Item's own state history; a persistence
service (which strategy — not every site persists every Item); the engine's own memory
(empty after a restart); or a defined "unknown" state with defined behaviour. This is a
safety choice, not an implementation detail — it decides whether a compressor's cooldown
survives a restart, and whether a duty-cycle guarantee can be proven at all in the first
minutes after one.

**I1 (cross-cutting) — which way the level dependency runs.** `define-energy-levels`'
_Level derivation_ escalates the current level on live PV surplus; this change's _Central
periodic evaluation_ says every algorithm in a cycle is judged against one snapshot, and
surplus is in that snapshot. As written the two are mutually recursive and neither change
mentions the other. Options: (a) the site level is an **input** handed to the engine,
computed by the level plane from its own readings — then the level and the snapshot can
disagree inside one cycle; (b) the level is a **product** of the cycle's own readings —
then the level plane is a function the engine calls, not a source it reads. The prototype
had to take (b) to keep "one snapshot per cycle" true, and reports it as the removal of an
inconsistency rather than an answer. Recorded in both changes' design notes.

## 5. Precedence between protections and the limit floor

Two requirements carry "regardless" clauses that can collide: the duty-cycle guarantee
switches a cooling device back ON "regardless of price or surplus" (participant model),
and the limit floor trims loads "regardless of what the plan proposed". If honoring a
max-OFF guarantee would exceed the electrical budget, which wins? Physics says the limit
floor; the fridge then runs as soon as headroom exists. Proposed ladder (to confirm):
**electrical limits > device protections > level gates > optimization** — matching the
reference's priority ordering, but never discussed in the thread, so flagged as an open
design decision rather than encoded as a requirement.

**N5 — the master stop makes this a three-way collision.** Nothing says what a protection
should do while the stop is engaged. The two are different kinds of safety statement: a
duty-cycle guarantee is _device_ safety ("this compressor must run within the hour"); the
master stop is _operator_ safety ("this system writes nothing now"). Options: the stop
outranks everything, protections included, and device safety reverts to whoever owned it
before the EMS; protections are the one exception that survives a stop; or the stop is
itself layered (stop optimization / stop everything). **This one must not be settled by an
implementation.** Both wrong answers are real hazards: subordinating the stop leaves an
operator unable to stop the engine writing; subordinating the protection can leave a
cooling device off indefinitely while the engine believes it is not in control. The
prototype subordinates protections to the stop — because a stopped engine dispatches
nothing at all, not because that reading was chosen.

## 6. Does the master stop halt evaluation, or only dispatch?

**N4.** _Master stop_ says "stop all engine **actuation**". Nothing says whether the tick
keeps running. In the prototype a stopped engine still invoked every contributed algorithm
on every cycle — and a contributed algorithm is arbitrary third-party code holding its own
openHAB access, so "stopped" stopped the engine's writes and not necessarily the site's.
Options:

1. Stop halts dispatch only — evaluation continues, so an operator can still see what the
   engine would do while it is stopped (which is what shadow already gives, see §2).
1. Stop halts evaluation as well — nothing contributed runs, and the exposure disappears
   along with the observability.
1. Two controls, one for each.

The prototype's report calls halting "both the safer and the cheaper reading". That is its
opinion, recorded and deliberately **not** adopted here: it is a maintainer decision about
what a kill switch means, and (1) versus (2) is also the difference between a stop you can
diagnose and one you cannot.

## 7. What survives a stop and resume

**E10.** "Disengaging it resumes normal operation" does not say whether the engine resumes
_as if it never stopped_ or _from a clean slate_. The state that differs, item by item:

- outstanding acknowledgement windows (the prototype resets them on stop — a choice, not a
  reading of the requirement);
- protection timers, which behave differently depending on §4's N3 answer: derived from
  device state they never stopped, held in engine memory they may be gone;
- whatever shadow-mode comparison state exists (see §14);
- loads deferred by the limit floor in the cycle the stop landed in (see §8).

Options: define it per state item; declare a clean slate and require the engine to rebuild
what it can from device state; or declare continuity and say how the gap is accounted for.
Note the interaction with §6: if the stop halts evaluation, a long stop makes "as if it
never stopped" unsupportable for anything time-based.

## 8. Trim or defer — which, and when

**E13.** The requirement now demands that the choice be made by a defined rule and be the
same on every run. The rule itself is open. What it has to answer:

- a load that cannot be trimmed at all (a plain switch) — is deferral its only option, and
  does being untrimmable make it cheaper or dearer to shed than a continuous load?
- a continuous load that partly fits — trim to the boundary, or defer whole?
- a declared minimum (a wallbox's 6 A) — trimming below it is not trimming, it is deferral
  under another name, so the minimum decides the outcome rather than the rule does;
- a Batch programme mid-run, which the taxonomy says may not be interrupted — deferral is
  not available, so either the rule or the taxonomy has to give;
- and, underneath all of it, whether trim and defer are the only dispositions the rule may
  reach for at all. The bullet above already shows the pair coming up one short for one
  class, and §17 (E16) asks whether a refusal may travel back to the planner as a
  re-plan — a third outcome that is neither a trim nor a deferral. Leaving a load untouched
  and swapping one load against another are two more. The requirement therefore asks the
  rule to settle _what happens_ to each affected load, not to choose between two named
  options.

Candidate shapes: priority order alone decides, and trim-versus-defer falls out of each
load's declared shape; an explicit per-profile-class rule; or a per-participant preference
declared alongside the load. The prototype trims to the boundary unless that breaks the
declared minimum and then defers — one reading among several, and it is not a proposal.

## 9. What the floor books: estimates or measurements

**E14.** The participant model treats a command as an envelope, not an order — which is
why a measurement exists at all. But within a cycle the floor has to allocate headroom
against draw that has not happened yet, so it books an _estimate_; the measurement arrives
cycles later and will differ. Unrecorded: which quantity the floor books; how the error is
reconciled on the next cycle (re-book against measurement, carry a reserve, or ignore); and
what happens when measurement persistently exceeds the booked estimate, i.e. a device that
ignores its envelope. This is also where two participant-model questions become
engine-visible: N1 (may a consumer declare its own measurement, and does the engine prefer
it over an estimate?) and E4 (a Simple consumer has no rated power) — the floor cannot book
an estimate for a participant that declares no power figure at all.

## 10. Mode changes and the budget

**E5.** A ModeControllable mode is an ordered label with no power semantics, so the floor
has nothing to enforce a budget against when an algorithm proposes a mode change — the one
participant class whose decisions are invisible to the limit floor. Options: a mode carries
an expected draw (declared with the mode, or learned — wave 4); mode changes are exempt
from the budget and the floor catches the consequence a cycle later through measurement; or
a ModeControllable consumer declares one envelope covering all its modes. The prototype
needed an `unknownDemandPolicy` knob here, which is what an open question looks like in
code, not a proposal for how to close it.

## 11. What counts as surplus

**E7.** Both this change and the participant model speak of surplus "derived from" grid
export, and no requirement says more. Undecided, concretely:

- does a charging battery's power count as surplus available to a consumer, given that
  charging it is a decision rather than a fixed load?
- does curtailed PV — production the site is deliberately not making — count as available?
- does an already-running managed consumer's own draw count, the recursion every surplus
  algorithm hits?
- is surplus instantaneous, averaged over the cycle, or forecast over the next one?

The prototype uses grid export only. Related: `define-participant-model`'s E6 — surplus
cannot be computed at all until the sign convention is defined for every provider role and
for setpoints, not only for the grid reading.

## 12. Decision outcomes — the corpus has no vocabulary

**N6.** _Shadow mode_, in its "Side-by-side validation" scenario, asks a user to compare
what the engine would have done against what their automation did, and _Deterministic
conflict resolution_ produces
decisions that lose. Nothing in the corpus names what became of a decision: applied,
superseded, deferred, suppressed, withheld, shadowed, stopped are all user-visible
behaviours and none of them is a defined term. The prototype invented seven values because
it could not report a cycle without them. Open: whether there is a defined outcome
vocabulary at all, and if so whether it is part of the API (an event, a status carried on a
decision, a REST resource) or an implementation's private business. Whoever ships first
settles it de facto — which is the argument for answering it now rather than later.
Related: §13 and §14 both need an outcome name to be answerable.

## 13. Decisions naming an unknown participant

**E15.** A script with a typo addresses a participant that does not exist. Nothing says
what happens. Options: reject the decision and log; ignore it silently; or surface it as an
algorithm error on a defined surface — which does not exist yet (`define-extension-points`
B11 asks the same question for a declaration in error, and finds no surface either). Note
the asymmetry: silent rejection is safe for the site and invisible to the author, so a
typo'd script looks like a working one that never fires. The prototype rejects and logs.

## 14. Shadow output — what surface, and at what rate

**E18.** _Shadow mode_ already names logging, so extending it is a decision rather than a
clarification, and its own _Side-by-side validation_ scenario is not testable against what
it names: at a one-minute tick the same decision is logged for hours, and a log is not
something a user compares against a week of their old automation. Options:

1. logs only, deduplicated so a decision is emitted when it changes;
1. structured events on the event bus, leaving persistence to whoever wants it;
1. a persisted series per participant, so the comparison is the same comparison any two
   series get;
1. Items the engine writes — which contradicts "without writing to any item" unless shadow
   output is explicitly exempted, and that exemption is itself a decision.

Related: §12 (an outcome needs a name before it can be published) and `define-energy-levels`
L23 (what publication means under shadow, for the level plane's own Items).

## 15. Algorithm identity and ownership

**I6.** The declaration plane worked ownership out carefully — contributor ownership,
precedence, user selection (`define-extension-points`). The algorithm plane has no
equivalent statement. Registering an algorithm under an id and replacing silently is
exactly what a reloaded script needs, and exactly what lets one script take over another's
id unnoticed. Open: how an algorithm id is formed; whether replacement is permitted and by
whom; whether a script reload is distinguishable from a collision (the same question
`define-extension-points` B14 asks about declarations, where a script cannot reliably know
it is being unloaded); and whether more than one algorithm may be active at once — the
corpus says "the algorithm", singular, throughout, and never states it.

## 16. Is the engine-owned enumeration complete?

**N7.** _Replaceable algorithm_ now says the enumeration is stated in one place and stated
to be complete; what belongs in it is open. Two candidates that `define-participant-model`
states in engine language and this change's list omits: the **level gate** ("the engine may
switch it ON … and does not steer it" for the `never` setting) and the **readiness
interlock** ("gates any engine-initiated start"). Options: both are engine-owned and the
list grows; both are inputs an algorithm is expected to honour and the list is already
right; or they are engine-owned for the default algorithm and overridable by a contributed
one — which is why "complete" is a statement about the list's closure and not about every
member being absolute: a complete enumeration may still qualify a member. This is the
security boundary of the whole extension story — it decides what a contributed script is
allowed to switch on, including a device its owner marked hands-off. Flagged for
maintainers; related to `define-participant-model` A1, which asks whether the gate belongs
on every consumer class in the first place.

## 17. Deferral and replanning

**E16.** The prototype found `expected-boiler-control.csv` attributed to
`define-grid-constraints` _and_ to this change's _Deterministic conflict resolution_, and
could not reproduce it from anything this change requires: the vector needs look-ahead
rescheduling, a runtime floor can only defer a load rather than move one to a cheaper hour,
and conflict resolution is about two decisions on one device in one cycle. The attribution
is a fixtures matter and is corrected there; the requirement gap is this change's. Open:
does the engine contract gain a deferral → replanning path — the floor reports what it
refused and the planner re-plans against that — or is a deferral terminal for the cycle,
with replanning entirely `define-grid-constraints`' business?

## 18. How the site level reaches the engine

**I2 (cross-cutting).** `define-energy-levels` specifies the level plane as _publishing_
Items — a current-level Item and a future-timestamped `TimeSeries` — while this change has
the engine _consuming_ a level. Under shadow-only the engine may not write anything, so
publication cannot be the path, and the corpus describes no other. Options: (a) an
in-process seam from classifier to engine, with publication a separate observable side
effect; (b) the engine reads the published Item, which couples the two planes through the
item registry and makes the level plane a hard runtime dependency of a cycle; (c) both,
with the Item as an observable mirror of an in-process value. These are materially
different architectures, not an implementation choice. Recorded in both changes' design
notes; related to §4's I1 (input or product) and §14 (what shadow may publish at all).

## 19. Priority direction is the participant model's to define

**E1 / A4.** _Deterministic conflict resolution_'s "the higher-priority decision is
applied" reuses the participant model's word without asserting a direction, and this
change's own source note — "lower runs first, higher wins on conflict" — points both ways
in one sentence. Deliberately not fixed here: the scale belongs to _Priority_ in
`define-participant-model`, and its direction, default and tie-break are being sharpened
there. This note exists so that a reader of the engine contract does not read a direction
into "higher-priority" that no requirement states. Related: `define-energy-levels` L1 —
wave 1 currently carries two ordinal scales (priority and level codes) pointing opposite
ways, neither stated in prose.

## 20. Per-phase enforcement can see less than the scenario implies (E2, E3)

_Electrical limits outrank optimization_ carries the "Phase-aware headroom" scenario, and
two prototype findings bear on it. Both are owned by `define-participant-model`; they are
noted here because this is where the scenario lives and where the shortfall shows.

**The declaration now exists (E2).** The scenario opens "a consumer that declares which
phase(s) it draws on", and at the time the prototype was built nothing in the participant
model could declare a phase — the prototype had to carry the assignment in engine
configuration purely to make the scenario testable. `define-participant-model` has since
gained a _Phase declaration_ requirement for exactly this dependency. What a phase is
called, what omitting one means, and whether providers take a per-phase shape are open
there (`define-participant-model` design.md §5.12).

**Per-phase _measurement_ is the half that is still missing (E3).** Accounting for a phase
needs to know that phase's load, and providers declare a single aggregate power reading,
so the uncontrolled load on a given phase is not observable. An engine in that position
can constrain only what it dispatches itself — a real limit on this scenario, since the
phase nearest its limit is usually loaded by something the engine does not control.

Open, and deliberately not answered: does per-phase enforcement cover controlled load only
— in which case the scenario should say so — or does the corpus owe a per-phase provider
shape? The first reading is cheaper and weaker; the second makes phase-aware headroom mean
what a reader of the scenario would assume it means. Related: §9, where the same
estimate-versus-measurement question arises for the budget.

## 21. Provenance recap

Thread-sourced: central evaluation, conflict resolution, limits (budget/phase), shadow
mode, replaceable algorithm. Reference-sourced (flagged inline): master stop, ACK
actuation, and the "regardless of algorithm" limit generalization. Reviewers should
treat the flagged three as field-proven proposals, not thread consensus.

Prototype-sourced (this pass, §§1–20 where a defect id is named): the wave-1 prototype
build, not the thread — see `docs/PROTOTYPE_FEEDBACK.md`. It changed three requirements,
each carrying the provenance in its Source line: the limit floor must state its
trim-or-defer rule (E13), the acknowledgement window must be bounded (E8), and the
engine-owned enumeration is exhaustive and authoritative (N7). No open question was
answered by that pass, no prototype behaviour was adopted as a default, and E11 (cadence)
was rejected as a re-report of §1.
