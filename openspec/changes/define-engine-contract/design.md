# Design notes — open questions

Sections 1–5 predate the wave-1 prototype and now carry its findings underneath them;
sections 6–21 were opened by it outright. Every paragraph that names a defect id
(E\*, N\*, I\*) is prototype-sourced, not thread-sourced — see `docs/PROTOTYPE_FEEDBACK.md`.
Where the prototype had to behave one way in order to compile and run, that behaviour is
reported as a build artefact, never as a proposal.

**Answered questions carry an ANSWERED block.** On 2026-08-02 the owner of the reference
implementation answered most of them (`docs/OWNER_DECISIONS.md`). Those are **owner
decisions in a reference implementation, not thread consensus**, and the requirements they
changed say so on their Source lines. Every option that was _not_ chosen stays in the
section below its answer, on purpose: a maintainer who engages later can overturn a
decision by pointing at a preserved alternative, and the cost is rewriting one requirement
rather than reconstructing the argument.

## 1. Evaluation cadence

Kai suggested pull-based scraping "possibly up to every minute", with push "in a future
iteration" ([1482078918](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1482078918)).
The reference runs a short fixed tick plus a debounced re-evaluation on relevant item
changes. Open: fixed tick only, tick + event-responsive, and what the default cadence is.
The requirement stays cadence-neutral ("regular cadence") until decided.

The prototype re-reported this as **E11** after picking 60 s for itself. Nothing changed
here: the question was already recorded, and the 60 s is a build artefact.

**ANSWERED — the owner, 2026-08-02 (D17, Part B row EC-1, `docs/OWNER_DECISIONS.md`), as a
stated default rather than a mechanism.** Both halves of the shape: **a tick, plus
event-responsive re-evaluation on the readings participants declare**. The shipped values
are **60 s for the tick and 5 s for the debounce**, and both are configuration — a site
that dislikes either changes a number, which is the whole reason this row sits in the
parameter pass rather than in Part A. The requirement now states them and carries a
scenario that a configured cadence is honoured, so "default" is testable rather than
decorative. Evidence on each half: the pull side is Kai's own "possibly up to every
minute", which fixes 60 s as the slowest defensible tick; the event side is production
experience — the reference runs a 5 s tick with a ~1 s debounce precisely because a control
that waits a whole interval reads as broken to the person standing at the appliance. The
60 s tick with an event trigger is the cheaper way to buy the same responsiveness.

Note the interaction the number carries, so a maintainer who changes one changes both
knowingly: the acknowledgement window (§3) is 60 s because the invariant behind it is
**two to three evaluation cycles**. At a 5 s tick that invariant would read 15 s. Whoever
moves the tick should move the ACK default with it.

_Not chosen, preserved:_ (a) **a fixed tick only**, no event path — the simplest engine to
reason about and to test, and the one whose worst-case latency is a stated number; its cost
is that the worst case _is_ the tick, so at 60 s a user watching a device sees a minute of
nothing. (b) **event-driven only**, no tick — maximally responsive and it does no work when
nothing moves; it never re-evaluates on time alone, so a plan that becomes wrong because
the clock advanced (a slot boundary, a deadline) is not noticed until some reading happens
to change. (c) **a 5 s tick**, the reference's own — proven responsive on a live building;
it multiplies every cycle cost by twelve for a system whose inputs are mostly minute-scale,
and the prototype adopted 60 s under exactly that reasoning. (d) **an adaptive cadence**
(fast while dispatching, slow while idle) — best of both on paper; it makes "the same
inputs always resolve the same way" harder to state, because when the engine looked becomes
part of the input.

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

**ANSWERED — the owner, 2026-08-02 (D3, `docs/OWNER_DECISIONS.md`).** Two controls, not
one. Shadow is compute-and-report-but-write-nothing and is settable globally, per
algorithm and per participant; the stop is a global boolean that halts everything (§6). A
fresh install is **shadowed, with the stop disengaged**, so the engine reports from its
first cycle without touching a device. Evidence: the reference binding ships exactly this
two-control shape (bridge `shadowMode=true` by default plus a per-controller
`shadowMode()`), and that is what staged its live cutover. E9 dissolves with it — the two
defaults no longer coincide, because they are no longer the same control.

_Not chosen, preserved:_ (a) **one unified control**, the shape the question opened with —
simpler API, one thing to explain, and the reference's own first cut; its cost is that a
user can never say "stop writing but keep showing me what you would do", and shadow's
default then collides with the stop's. (b) **global-only shadow** — no per-algorithm or
per-participant sets; cheaper, and it removes the question of what a participant in shadow
does to an algorithm that expects it to move; its cost is that a partial cutover (one
device live, the rest observed) becomes an all-or-nothing decision.

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

**ANSWERED IN PART — the owner, 2026-08-02 (D8, and D17 for the window length,
`docs/OWNER_DECISIONS.md`).** Two of the four sub-questions above are settled, and the
requirement now states both:

- **What counts as an acknowledgement** (3): the control item's state matching the
  commanded value under a unit-aware comparison, and a **tolerance band counts** where the
  participant declares one. Grounded in the owner's live hardware — OCPP chargers report
  lagging, imprecise values, which is why the binding has an ACK window at all. The
  15.999 A question the section opens with therefore has an answer, and it is
  per-participant rather than a number the specification invents.
- **How long the window is** (1): **60 s engine-wide, overridable per participant**, not
  learned — the parameter pass (D17, row EC-3a). The invariant behind the number is
  two-to-three evaluation cycles.
- **What expiry means** (2): the outstanding command **lapses and actuation resumes**, the
  participant being reported as unacknowledged. Grounded in production: a charger that
  silently resets its current after a fault cycle is only recovered by re-sending.

_Not chosen, preserved:_ the **safety-conservative reading** — a device that never
acknowledges is withheld from further commands until something clears it, quarantined and
surfaced as an error rather than commanded blind. It is the better reading for a device
whose state genuinely cannot be observed; its cost on real hardware is that every
feedback-less `autoupdate="false"` device becomes permanently unmanaged after one command,
silently. Also preserved: the third reading the sharpening named, **never re-send, never
give up, block indefinitely** — a change to the requirement rather than a configuration of
it. Also preserved: **exact equality only, with no tolerance mechanism at all**, which
needs no per-participant number and refuses an off-target device outright.

**What a participant declares is now stated, and where.** The window and the tolerance band
are both declarable per participant, in `define-participant-model` _Acknowledgement
declaration_ — which is where the corpus enumerates what a declaration may carry, and which
previously carried neither although this requirement depends on both. The band is fixed as
an **absolute quantity in the control Item's own dimension** rather than a fraction of the
commanded value. That is not a decision the owner was asked for; it is the one thing the
15.999 A example cannot be read without, since 0.01 A and 0.1 % differ by two orders of
magnitude on it. _Preserved:_ a **proportional** band, which scales across a fleet of
differently-rated devices without a per-device number and is what a percentage-minded
implementer would reach for first; its cost is that the same declaration means 0.032 A on a
32 A charger and 0.006 A on a 6 A one, so the tolerance shrinks exactly where the device is
least precise.

**Still open, and deliberately not read into the answer:** where acknowledgement handling
_lives_ (the paragraph this section opens with — core engine, per-device adapters, or core
with an opt-out for protocols carrying their own acknowledgements, which is what the pack
recommended); **the default tolerance value**, which the owner recorded as still to be set
and which the requirement therefore leaves per-participant with no default; sub-question 4,
**what happens to a changed command** — the pack recommended "a changed command supersedes
immediately, only an identical repeat is suppressed, and a decrease is never suppressed",
and that recommendation was not put to the owner; and, a case that recommendation would also
close, **what the electrical-limit floor does when it must reduce a load whose previous
command is still unacknowledged**. The strict reading of the requirement is already safe — a
reduction is a _different_ command, and only re-sending the _same_ one is forbidden — but
the corpus states that nowhere, and the failure mode if an implementer reads it the other way
is a floor that cannot shed for a whole acknowledgement window. The pack's "a decrease is
never suppressed" is the one-clause close whenever a maintainer wants it.

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

**ANSWERED — the owner, 2026-08-02 (D7 and its refinement, `docs/OWNER_DECISIONS.md`).**
Elapsed time comes from the Item's own state history, read through
**`Item.getLastStateChange()`**; where that is null the engine **starts the clock at first
observation** and reports the participant as protection-unknown; and **the engine keeps no
protection timers of its own**. Core already answers the mechanism:
`Item.getLastStateChange()` exists and `PersistenceManagerImpl.restoreItemStateOnStartup`
restores it explicitly — so a compressor's cooldown survives a restart **iff the Item is
persisted with `restoreOnStartup`**. That contingency is why the answer lands as two
requirements rather than one with a note: _Protection timing comes from device state
history_ and _An unpersisted protection is reported_, the second so that a guarantee
degraded by the site's persistence configuration is visible instead of assumed. This is a
**departure from the reference binding**, whose `constrainOnOff` leaves the desired state
untouched when the elapsed time is unknown. Related: `define-participant-model` §5.15,
where the "forced restart" is restated observably as any OFF→ON transition the engine did
not command.

_Not chosen, preserved:_ (a) **engine memory only** — no dependency on the site's
persistence at all, and trivially correct while the engine runs; its cost is that every
restart silently voids every duty-cycle guarantee. (b) **engine memory with unknown treated
as a full wait** — fail-safe in the protection's own direction, never under-waiting; its
cost is that every restart idles every protected device for a full cooldown. (c) **a
persistence query per cycle** instead of the Item's own `lastStateChange` — works on a site
that persists everything; its cost is a hard dependency from a safety function onto a
queryable persistence service, and a strategy the requirement would then have to name.

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

**ANSWERED — the owner, 2026-08-02 (D6, `docs/OWNER_DECISIONS.md`).** Option (b): **the
engine computes the level from its own snapshot**, the level plane being a pure function it
calls rather than a service it reads. That is what makes "one consistent snapshot per
cycle" true rather than nearly true — level and readings cannot disagree inside a cycle —
and it also forecloses one of §18's options, since a published Item cannot be the transport
for something the engine derives itself. Note honestly that the prototype reported (b) as
the removal of a contradiction, not as an answer; the answer is the owner's.

_Not chosen, preserved:_ (a) **the level is an input** computed by the level plane from its
own readings and handed to the engine — closer to masipila's planned-versus-current split,
and it keeps the level plane independently useful to anything that is not the engine; its
cost is that the level and the snapshot can disagree within one cycle, which is invisible
in tests and shows up on partly-cloudy days. If (a) is ever adopted, publication becomes an
_output_ of the engine rather than its input, and §18's option (b) returns with it.

**Still open:** E17 above — whether an algorithm may retain a snapshot beyond its cycle,
and therefore whether the snapshot is part of the public API. The pack recommended an
immutable, explicitly retainable snapshot carrying its own timestamp, with a stated minimum
content; that recommendation was not put to the owner and is not read into D6.

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

**ANSWERED — the owner, 2026-08-02 (D2 for the ladder, D1 for the stop's place above it,
`docs/OWNER_DECISIONS.md`).** The proposed ladder is confirmed as written —
**electrical limits > device protections > level gates > optimization** — and the **master
stop sits above all of it, protections included**. It is now a requirement of its own
(_Constraint precedence ladder_) rather than a note, so nothing else in the corpus has to
restate it and no algorithm or configuration can reorder it. Evidence, independent of the
pack: the reference's `constrainOnOff` javadoc says it outright — "the fuse outranks the
freezer, deliberately" — and physics agrees, since exceeding the fuse trips the board
including the fridge. N5 resolves with it: while the stop is engaged, device safety reverts
to whoever owned it before the EMS, and the requirement carries the rider that this
non-enforcement is **reported**, not merely logged.

**The ladder is absolute except where two other decisions put something on its rung, and
the requirement now says so.** D9 exempts a **Batch programme mid-run** from the
electrical-limit floor, and D13 lifts `never` onto every consumer class; D14 repeats the
Batch exemption for the safe state. Non-interruptibility is a device protection, and a
hands-off flag is a user prohibition, so both of those exemptions place something **above**
the electrical-limit rung — precisely the inversion this ladder forbids, and precisely
preserved alternative (a) below. Left implicit it was a genuine fork: an implementer reading
the ladder interrupts the dishwasher to save the fuse, one reading the floor lets the
breaker trip and takes the fridge with it. _Constraint precedence ladder_ therefore states
both exceptions in the ladder itself, as loads the rung **books** rather than sheds, with a
refusal-and-reason where the remainder still does not fit — so the ladder's own guarantee
("nothing may reorder these") stays checkable against the corpus rather than being quietly
false.

_Not chosen, preserved:_ **an electrical limit may interrupt a running Batch and may shed a
hands-off load** — the ladder stays absolute with no exceptions to state, which is the
simplest thing to test and the reading physics argues for, since a tripped breaker takes the
dishwasher down anyway and mid-cycle. Its cost is that the taxonomy's own defining promise
for the Batch class ("the engine never interrupts it before the program completes") becomes
conditional, and a half-washed load is a real user harm the corpus chose to weigh. It is the
option to reach for if the exception list ever grows a third member.

_Not chosen, preserved:_ (a) **protections above electrical limits** — device safety first,
which is the reading a fridge owner would pick; its cost is that a configuration file can
force the engine past the fuse, and the resulting trip takes the fridge with it anyway.
(b) **protections survive the stop** as the one exception — device safety continues while
operator safety is asserted; its cost is that the operator cannot make the engine stop
writing, which is the one thing a kill switch is for. (c) **a layered stop** (stop
optimization / stop everything) — expressive, and it would let an operator keep protections
running while halting the rest; its cost is that "stopped" stops meaning one thing, and the
control most likely to be reached for in a hurry becomes the one needing a choice.

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

**ANSWERED — the owner, 2026-08-02 (D1, `docs/OWNER_DECISIONS.md`) — and this one
overrode the recommendation put to them.** Option (2): **the stop halts everything,
evaluation included**, and no contributed algorithm is invoked while it is engaged. Both
readings and both arguments, because this is the most consequential disagreement in the
set:

- **The owner's argument, adopted.** A stopped engine that still runs third-party
  algorithms has stopped the engine's writes, not the site's — a contributed algorithm is
  arbitrary code holding its own openHAB access. Relying on add-ons to no-op when told the
  engine is stopped is exactly the assumption a kill switch must not make. So "stopped"
  means nothing of ours runs.
- **The recommendation, not adopted, preserved in full.** Option (1): the stop halts
  **dispatch only**; evaluation continues and `stopped` is exposed in the context so a
  well-behaved contributed algorithm can no-op of its own accord. Its argument: _a stopped
  engine is a black box exactly when you need to know why you stopped it_ — the operator
  loses the ability to see what the engine would be doing at the moment they most want to
  know, and shadow mode, which already provides observation without writes, is then the
  only diagnostic left. It also treats the exposure as a contributor-contract problem
  rather than a runtime one.
- **Option (3), two controls — one for evaluation, one for dispatch** — also preserved. It
  serves both cases at the cost of a third control on the safety surface and of a state
  ("evaluating but never dispatching") that is indistinguishable from shadow.

Consequence to keep in view: with evaluation halted, §7's "resume as if it never stopped"
is unsupportable for anything time-based, which is exactly why protections are read from
device state history (§4) rather than from engine memory.

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

**ANSWERED IN PART — the owner, 2026-08-02 (D1, and D17 row EC-7,
`docs/OWNER_DECISIONS.md`).** D1 settles the frame: the stop halts evaluation, so
continuity is not on the table for anything time-based, and the answer is per state item
rather than a single slogan.

- **Protection timers** continue by construction — they are not engine state at all, but a
  reading of the device's own history (§4). This is the item that would otherwise have made
  a long stop dangerous.
- **Acknowledgement windows** are cleared: engine-held ephemeral state resumes from a clean
  slate (D17, row EC-7; the reference already calls `SetpointDedupe.forget(key)` when the
  engine stops owning a device).
- **Deferred loads and shadow comparison state** are not settled by these decisions and
  stay open here.

_Not chosen, preserved:_ **continuity** — the engine resumes as if it never stopped,
accounting for the gap explicitly; it is the reading that makes a brief operator stop
invisible to the site, and it is unsupportable for time-based state once evaluation halts.
Also preserved: **a whole-engine clean slate**, rebuilding everything it can from device
state, which is simpler to reason about but throws away the acknowledgement state of a
device commanded moments before the stop.

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

**ANSWERED — the owner, 2026-08-02 (D9, with the disposition set from D16 / pack A6,
`docs/OWNER_DECISIONS.md`).** The first candidate shape: **priority orders, the load's own
shape decides how**, and no new user declaration is needed. Point by point against the
bullets above:

- a load that cannot be trimmed (a plain switch) defers, and **untrimmability never changes
  the shed order** — it decides what happens to a load, not when its turn comes;
- a continuous load trims to the largest value at or above its declared minimum;
- trimming below a declared minimum is deferral under another name, so the minimum decides
  the outcome, exactly as the bullet observes;
- a **Batch programme mid-run is exempt**: it is left untouched and its draw is re-booked
  as load the others are trimmed against. The spec's warning that "either the rule or the
  taxonomy has to give" is settled **in the taxonomy's favour**;
- and the dispositions are a closed set of four — **trim, defer, leave untouched, refuse
  with a reason** — which answers the "one short" problem the bullet raised without
  opening the rule to arbitrary outcomes.

**Two things the rule left unstated, made definite by reconciliation rather than by D9.**
First, **what deferral means for a load that is already running.** Simple is unconditional
in the disposition table, and the floor books running loads, so a running fridge can be
dispositioned — but everywhere else in the corpus "defer" postpones a _start_, and
`define-grid-constraints` _Look-ahead and replanning live with the planner_ consumes
deferrals as loads to reschedule. The requirement now says it: deferring a running Simple
load is a **switch-off**, subject to device protections, and the resulting deferral is
replannable exactly as a postponed start is. _Preserved:_ the reading in which the floor may
only ever postpone a start, so a running Simple load is untouchable and the floor's only
remaining move against it is to refuse — cheaper to reason about, and it leaves the floor
unable to resolve an overload made entirely of running switches. Second, the **hands-off
exemption**, which D13 created after this section was written and which now sits alongside
the Batch one; see §5.

_Not chosen, preserved:_ (a) **a per-participant declared preference** ("trim me, don't
defer me") — most expressive, and it lets a user encode knowledge the model cannot infer;
its cost is another declaration on every consumer, and a preference that has to be
overridden the moment it conflicts with the load's own minimum. (b) **an explicit
per-profile-class rule** written in the spec instead of derived from shape — easier to
tabulate; it is nearly the same outcome as the chosen rule, and it stops being right the
moment a class gains a device that does not fit the class's stereotype. (c) **a deferral
that travels back to the planner as a re-plan** — the third outcome §17 asks about; it
stays open there and is not part of this answer.

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

**ANSWERED — the owner, 2026-08-02 (D15, `docs/OWNER_DECISIONS.md`).** The floor books
**`max(declared, measured)` for anything already running and the declared figure for
anything not yet started**, and it **re-books from scratch every cycle with no reserve**.
The reconciliation question dissolves rather than being answered: with no carried state
there is no error to reconcile, and a device that persistently exceeds its envelope is
constrained because the measurement is what gets booked. Both participant-model questions
close with it: N1 becomes the _Declared power measurement_ requirement (a participant may
name an Item carrying its live power, preferred over the estimate except when admitting a
load that is not yet running) and E4 becomes the per-class figure in _Consumer power
figure_, with `ratedPower` **optional** on Simple so live declarations stay valid and the
absence reported as a gap rather than a rejection.

_Not chosen, preserved:_ (a) **book estimates only** — one number per participant, no
dependency on a measurement Item; its cost is that "a command is an envelope, not an order"
becomes unverifiable and an overshooting device stays invisible. (b) **measure only** —
truthful about the present; it can never reserve room to _start_ anything, which is the one
allocation the floor exists to make. (c) **carry an error term or a reserve between
cycles** — smooths a noisy site; it introduces a tuning constant nobody can derive and
falsifies "the same inputs always resolve one overload the same way", which the requirement
above depends on. (d) **reuse the Simple on-threshold as the rating** — needs no new
declaration and is what the prototype did; it silently over-books, always in the same
direction, because thresholds are set with margin.

## 10. Mode changes and the budget

**E5.** A ModeControllable mode is an ordered label with no power semantics, so the floor
has nothing to enforce a budget against when an algorithm proposes a mode change — the one
participant class whose decisions are invisible to the limit floor. Options: a mode carries
an expected draw (declared with the mode, or learned — wave 4); mode changes are exempt
from the budget and the floor catches the consequence a cycle later through measurement; or
a ModeControllable consumer declares one envelope covering all its modes. The prototype
needed an `unknownDemandPolicy` knob here, which is what an open question looks like in
code, not a proposal for how to close it.

**ANSWERED — the owner, 2026-08-02 (D15, `docs/OWNER_DECISIONS.md`).** The second option,
with the first available as an opt-in: **a mode change is exempt from the planner's
budget** and the runtime floor catches the consequence a cycle later through measurement,
while a declaration **may** carry a per-mode draw, which upgrades that mode into the
budget. The exemption is grounded rather than pragmatic: an SG-ready mode 3 draws whatever
the heat pump decides to draw, so a mandatory declared number would be fiction, and the
reference names the same limitation honestly ("mode loads' draw is unknown to the surplus
budget").

_Not chosen, preserved:_ (a) **a mandatory expected draw per mode**, declared or learned —
it makes every class visible to the floor and closes the one hole in budget enforcement;
its cost is a fabricated number for exactly the device class the mode model exists for.
(b) **one envelope covering all modes per consumer** — a single conservative figure, easy
to declare; it over-books the restricted modes permanently, which on a heat pump is most of
the day.

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

**ANSWERED IN PART — the owner, 2026-08-02 (D10, on the sign convention D11 fixes,
`docs/OWNER_DECISIONS.md`).** **Surplus is grid export plus the battery charging the engine
can reclaim.** Bullet by bullet:

- **a charging battery counts.** Charging it is a decision the EMS itself made, not a fixed
  load, so that power is available to a better-priority consumer. Grounded on the owner's
  own site: solar-first EV charging that ignored battery charging idled while the battery
  absorbed everything.
- **curtailed PV does not count.** It is capacity rather than power — a load switched on
  against it imports until the inverter ramps — and the reporting it would need is missing
  on most inverters.
- the remaining two bullets — **whether an already-running managed consumer's own draw
  counts** (the recursion), and **whether surplus is instantaneous, averaged over the cycle
  or forecast over the next one** — the owner recorded as **still open**, and they are not
  read into this answer.

E6 unblocks with it: the signs are fixed centrally in `define-participant-model` §5.11
(grid + = export, battery + = charging, PV + = producing, consumers + = consuming), which
is what makes "export plus reclaimable charging" an arithmetic rather than a phrase. The
normative statements live where the quantity is used — `define-energy-levels` _Surplus
escalation of the current level_ and `define-optimization-objectives` _Selectable
objective_ — rather than being restated a third time here.

_Not chosen, preserved:_ (a) **grid export only**, which is what the prototype used —
unambiguous, one signed reading, nothing to explain; a 3 kW battery charge then hides all
surplus and every surplus-driven load oscillates unless hysteresis is hand-tuned. (b)
**export + battery + curtailed PV** — the most complete picture of what the site could
absorb; it needs inverter curtailment reporting that many inverters do not provide, and it
credits power that does not exist until something ramps. (c) **the decision pack's
two-quantity split**, which the owner did not adopt: `surplus` = grid export only, and a
second named `allocatableBudget` = surplus + the draw of participants the engine is
currently steering, with the recursion answered by naming two quantities instead of one and
with a charging battery treated as a participant the allocator serves by priority rather
than as surplus. It is the option to reach for if the recursion bullet above turns out to
need an answer, and it is preserved in full because it answers a question this decision
leaves open.

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

**ANSWERED — the owner, 2026-08-02 (D16 / pack A8, `docs/OWNER_DECISIONS.md`).** There is
a defined vocabulary and it is **part of the API**: an outcome from a fixed enumeration —
applied, shadowed, stopped, superseded, deferred, suppressed, withheld, rejected — plus a
free-text reason on every decision, **published as events, deduplicated so an unchanged
decision is not re-emitted**, with the current cycle readable over REST and one
engine-published status Item summarising it. Core writes **no Item for an individual
decision**, which is what keeps "shadow writes nothing" literally true. Core precedent
rather than invention: `RuleStatus` + `RuleStatusDetail` + `RuleStatusInfoEvent` plus a REST
resource is the same shape, and core publishes events while leaving persistence to whoever
wants it.

_Not chosen, preserved:_ (a) **no vocabulary at all** — an implementation's private
business; it is the cheapest option and it makes every consumer of a shadow run
reverse-engineer log strings. (b) **core persists outcomes itself** — the comparison
becomes the same comparison any two series get; core would then own persistence policy for
a diagnostic, which it never does anywhere else. (c) **a status carried on the decision
object only**, with no event — enough for an in-process algorithm, invisible to rules and
UIs.

**The one member D1 nearly made unreachable — `stopped`.** This vocabulary comes from pack
A8, which assumed the pack's own **dispatch-only** stop: under that reading a stopped engine
still evaluates, so every cycle keeps producing decisions and every one of them is marked
`stopped`. D1 overrode that (§6) and halts evaluation, at which point a stopped engine
produces no decisions at all and the value has nothing left to describe. The two decisions
were taken in the same session and the collision is theirs, not the pack's. Resolved above
in the vocabulary's favour: _Master stop_ now says writes cease **immediately** and carries
a scenario giving `stopped` exactly one home — the decisions of the cycle in flight when the
stop landed, published and not written. It is a narrow home on purpose; the alternative is
narrower still.

_Not chosen, preserved:_ **drop `stopped` from the per-decision vocabulary** and carry the
stop only on the engine's own status Item. That is what the enumeration means if the
in-flight cycle is abandoned unpublished rather than published — arguably the more honest
model, since "the engine is stopped" is a property of the engine and not of any one
decision, and it removes a value an implementer can otherwise never produce in steady state.
Its cost is that the moment of stopping becomes unobservable per device: an operator asking
"what was in flight when I hit the switch, and did any of it land?" gets no answer from the
decision stream. A maintainer who prefers it deletes one enum member and one scenario.

Consequence recorded here so it is not read as a contradiction: _Shadow mode_ now says
"without writing to any **participant** Item" rather than "any item", because the engine's
own status Item is not a control write. A maintainer who wants shadow to write nothing at
all, status included, is overturning that one word.

## 13. Decisions naming an unknown participant

**E15.** A script with a typo addresses a participant that does not exist. Nothing says
what happens. Options: reject the decision and log; ignore it silently; or surface it as an
algorithm error on a defined surface — which does not exist yet (`define-extension-points`
B11 asks the same question for a declaration in error, and finds no surface either). Note
the asymmetry: silent rejection is safe for the site and invisible to the author, so a
typo'd script looks like a working one that never fires. The prototype rejects and logs.

**ANSWERED — the owner, 2026-08-02 (D16 / pack A8, `docs/OWNER_DECISIONS.md`).** The
decision is **rejected, with a reason, and counted**, on the surface §12 defines — so the
third option stops being hypothetical, because the surface now exists. The reference's
`PriorityScheduler.run` already does the analogue for a controller that throws: warn, skip
its requests, never take down the tick.

_Not chosen, preserved:_ (a) **ignore silently** — safe for the site and the cheapest to
implement; it is the option whose asymmetry the paragraph above names, and it makes a
typo'd script indistinguishable from a working one. (b) **reject and log only** — what the
prototype did; it is invisible to a UI and to any rule that would want to alert on it.

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

**ANSWERED — the owner, 2026-08-02 (D16 / pack A8, `docs/OWNER_DECISIONS.md`).** Option
(2), with (1) folded into it: **structured events on the event bus, deduplicated so a
decision is emitted when it changes**, leaving persistence to whoever wants it, plus the
REST view of the current cycle and the one status Item from §12. That makes _Side-by-side
validation_ testable — a week of events is comparable, a log line repeated every minute is
not.

_Not chosen, preserved:_ (1) **logs only, deduplicated** — no new API surface at all, and
it is what the requirement already named; a log is still not something a user compares
against a week of their old automation. (3) **a persisted series per participant** — the
comparison becomes the same comparison any two series get, which is the nicest end-user
story; it puts core in charge of persistence policy for a diagnostic. (4) **Items the
engine writes per decision** — the most visible option, and it contradicts shadow mode
unless shadow output is explicitly exempted, an exemption that is itself a decision.

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

**ANSWERED IN PART — the owner, 2026-08-02 (D4, `docs/OWNER_DECISIONS.md`).** The last
sub-question is settled: **several algorithms may be active at once**, and conflicts
between them are resolved by the same lower-number-wins priority rule that orders consumers
(_Deterministic conflict resolution_). The corpus's singular "the algorithm" was never a
statement, and the prototype ships two algorithms together with a test that depends on
both being active.

**Still open:** how an algorithm id is formed, whether replacement is permitted and by
whom, and whether a script reload is distinguishable from a hijack. The pack recommended
keying it on a contributor-supplied `contributorId` under the same precedence rule as
declarations; that half was not put to the owner, and D5 answers identity for
**participants**, not for algorithms. Note the neighbouring answers that do exist: core's
`providersupport` unload hook removes the "a reloaded script must replace itself" objection
(Part C, EP-5.9), and contributed-objective identity is first-registration-wins with a
duplicate refused (Part C, OO-6a) — both are evidence for whoever closes this, not the
answer to it.

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

**ANSWERED — the owner, 2026-08-02 (D13, `docs/OWNER_DECISIONS.md`).** The line is drawn
between **prohibitions and preferences**: prohibitions are engine-owned and the list is
closed; preferences are algorithm inputs. Engine-owned, in one place and complete —
evaluation, the electrical-limit floor, shadow, the master stop, actuation, plus the user's
prohibitions: the **hands-off flag**, the **user-declared per-consumer level gate**, the
**readiness interlock**, **device protections**, and **which readings count as safety
inputs** (a contributor may not claim that status for itself, and a forecast series is by
construction not one). Overridable, and the one member that is: the engine's **own**
level-derived steering, so a planner can still lift the engine's gate under deadline
pressure. Both of §16's candidates therefore go **in** the list. Grounded in the reference,
where gating lives in the asset handler below every controller — "the last line before an
item is written". Structural consequence carried into `define-participant-model`: `never`
lifts **off** the Simple profile onto the consumer, so all four classes can be marked
hands-off (§5.7 there).

_Not chosen, preserved:_ (a) **everything is an algorithm input**, the list staying as it
was — maximum freedom for a contributed planner, and the reading a scripting-first design
would pick; its cost is that a contributed script can start a device its owner marked
hands-off, which makes the most-cited use case advisory. (b) **everything is engine-owned,
including the engine's own level-derived steering** — the simplest boundary to explain and
to test; its cost is that no algorithm can lift a gate under deadline pressure, which a
real planner legitimately does. (c) **engine-owned for the default algorithm, overridable
by a contributed one** — the reading that treats "complete" as a statement about closure
rather than about absoluteness; it is the option a maintainer who dislikes the split should
reach for first.

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

**ANSWERED — the owner, 2026-08-02 (D16 / pack A14, `docs/OWNER_DECISIONS.md`).** The
second reading: **a deferral is terminal for the cycle.** This change owns the **runtime
floor and a deferral-to-report path only** — the floor refuses a load, publishes that
outcome as `deferred` with its reason (§12), and the cycle stays a pure function of its own
snapshot. **Look-ahead scheduling and the replanning of deferred loads belong to
`define-grid-constraints`**, whose _Look-ahead and replanning live with the planner_
requirement states it from that side and consumes those published outcomes on the planner's
own cadence. The two changes therefore say the same thing once each, from their own side,
rather than both claiming replanning or neither. E16's gap closes as a consequence: the
boiler vector needs look-ahead rescheduling, and look-ahead now has exactly one owner —
which is also why the fixture's attribution moves rather than the requirement growing.

_Not chosen, preserved:_ (a) **a deferral → replanning path inside the engine contract** —
the floor reports what it refused and the planner re-plans against that _within_ the cycle;
it is the more responsive shape and it removes a round trip, and its cost is that a cycle
stops being a pure function of its snapshot, since the plan then depends on what the floor
did to the previous plan in the same cycle. (b) **both changes describe replanning**, each
for its own layer — nobody has to look elsewhere; two requirements then own one behaviour
and drift apart on the first edit.

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

**ANSWERED — the owner, 2026-08-02 (D6, `docs/OWNER_DECISIONS.md`).** Option (a), and it
follows from §4 rather than being a second decision: the level plane is a **function the
engine calls in process**, so the level reaches the engine through an in-process seam and
publication is a separate, observable side effect. That is also the only reading compatible
with shadow mode, which writes nothing and must still gate on a level.

_Not chosen, preserved:_ (b) **the engine reads the published Item** — no new seam, and the
level plane's output is inspectable by anything in openHAB; its cost is a hard runtime
dependency on the item registry for a value the engine derives itself, a renamed Item
silently disabling level gating, and shadow mode being impossible. (c) **both, with the
Item as an observable mirror** — this is what the answer amounts to in practice, and it
stays preserved as the explicit form for whoever writes the level plane's publication
requirement; the difference is which one is normative as the transport.

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

**ANSWERED — the owner, 2026-08-02 (D4, `docs/OWNER_DECISIONS.md`).** **Lower number =
better priority, in both senses** — served first _and_ wins conflicts — with 100 as the
default and participant id ascending as a stateless tie-break. _Deterministic conflict
resolution_ above now says "better priority, which is the lower number" instead of
"higher-priority", so the engine contract no longer borrows a word whose direction was
unstated. The two ordinal scales are deliberately **not** forced to point the same way:
priority is a rank (lower is better, like every rank), the level scale is a magnitude
(higher means more available), and each is stated in prose where it is defined.

**Known follow-on, recorded rather than resolved:** the reference binding's live
`Controller` contract says "lower runs first, higher wins on conflict" and
`EmsManagerBridgeHandler` implements it (ascending sort, then last write lands). Adopting
lower-wins-conflict means reconciling the binding — this decision is a departure from
running production behaviour, not a description of it. The earlier claim that
`buildEngineDispatchSet` proved lower-wins was **wrong and has been struck**: that method
reads no priority number at all.

_Not chosen, preserved:_ (a) **higher number = better** — the reading OSGi's
`service.ranking` would suggest to a core reviewer; it inverts every fixture in the corpus,
and note that `service.ranking` governs _which provider is used_ rather than serving order,
so the two conventions are not actually in conflict. (b) **split directions** — lower runs
first, higher wins on conflict, which is what the reference binding says today; it is the
only option that requires no binding change, and it is a sentence that points both ways.
(c) **a stateful tie-break** (rotation or fair-share between equal priorities) — arguably
what a user expects "equal priority" to mean; it makes overload resolution
unreproducible from a snapshot and kills fixture conformance, and it belongs to an
algorithm that owns a priority rather than to the ordering primitive.

**Still open, and not settled by D4:** the tie-break between two _decisions_ of equal
priority addressing one device. D4 fixes the tie-break for ordering _participants_
(participant id ascending); nothing states what happens when two algorithms of equal
priority contradict each other on one device.

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

**ANSWERED — the owner, 2026-08-02 (D16 / pack A10, `docs/OWNER_DECISIONS.md`).** Both
halves, in the only combination that is honest: **per-phase enforcement covers controlled
load only**, and the scenario now says so — _unless_ providers declare per-phase readings,
which `define-participant-model`'s _Phase declaration_ now allows them to do (optional, so
a single aggregate reading stays valid). Phases are the **integer indices 1, 2, 3**; a
participant declaring none is **exempt from per-phase enforcement, still constrained by the
site total, and reported as a declaration gap** wherever the site declares per-phase
budgets. Grounded in production: the reference carries `totalAmpsL1/L2/L3` and computes
real per-phase headroom, which depends entirely on per-phase measurement existing.

_Not chosen, preserved:_ (a) **an undeclared consumer is attributed to every phase** —
conservative-sounding; it over-counts a single-phase load threefold and starves the site the
first time someone forgets a declaration. (b) **an undeclared consumer is attributed to one
unknown phase** — the worst of both, since the engine then constrains a phase the load may
not be on. (c) **mandatory per-phase provider readings** — makes the scenario mean what a
reader assumes; it invalidates every single-Item provider declaration in existence, which
is why the readings are optional and the shortfall is stated instead.

## 21. Degraded safety inputs and the safe state

**E12 (cross-cutting), filed against `define-extension-points` design.md §5.7 and answered
here because the runtime behaviour is this change's.** `extension-surface`'s _Graceful
degradation on contributor loss_ carries the scenario "Safety input lost — degrade safe,
not blind", which says the engine "falls back to a conservative safe state for
engine-steered loads (floor levels or pause)". Neither half had an operational meaning:
nothing said what makes a reading stale, and "floor levels or pause" names two different
behaviours. A faithful implementation of the corpus as it stood left the safety branch
inert.

**ANSWERED — the owner, 2026-08-02 (D14, `docs/OWNER_DECISIONS.md`).** **Freeze and
floor**: refuse every increase, floor Controllable loads at their declared minimum, drop a
ModeControllable load to its most restricted mode, switch Simple loads off — **all of it
subject to device protections**, so a Simple load inside its minimum runtime is held rather
than shed (the safe state may refuse increases immediately, but may only shed once the
ladder in §5 permits), and **never interrupt a running Batch programme**. Staleness is
**an optional per-participant maximum age plus unreadable / `UNDEF` / `NULL`, which always
trips**; no new ageing mechanism is invented, and a site may use core's `expire` namespace
to turn a frozen Item into `UNDEF`. Matches live behaviour on the reference site: when the
measurement bridge goes dead, every charger caps at its 6 A minimum rather than stopping.

_Not chosen, preserved:_ (a) **pause everything on a stale safety input** — unambiguous,
and the reading a safety-first review would pick first; its cost is that a five-second gap
in one reading stops the house, which is how a user learns to uninstall an EMS. (b) **keep
running on last known values** — no behaviour change at all, so nothing surprises the user;
it cannot tell a five-second gap from a five-hour one, which is the case the requirement
exists for. (c) **a staleness mechanism of the EMS's own** (an ageing service, a heartbeat)
— independent of how the site configures items; it duplicates `expire` and gives the
framework a second, competing notion of "this reading is old".

## 22. Provenance recap

Thread-sourced: central evaluation, conflict resolution, limits (budget/phase), shadow
mode, replaceable algorithm. Reference-sourced (flagged inline): master stop, ACK
actuation, and the "regardless of algorithm" limit generalization. Reviewers should
treat the flagged three as field-proven proposals, not thread consensus.

Prototype-sourced (§§1–21 where a defect id is named): the wave-1 prototype
build, not the thread — see `docs/PROTOTYPE_FEEDBACK.md`. It changed three requirements,
each carrying the provenance in its Source line: the limit floor must state its
trim-or-defer rule (E13), the acknowledgement window must be bounded (E8), and the
engine-owned enumeration is exhaustive and authoritative (N7). No open question was
answered by that pass, no prototype behaviour was adopted as a default, and E11 (cadence)
was rejected as a re-report of §1.

Owner-sourced (2026-08-02, every ANSWERED block above and every Source line that names a
`D`-number): **decisions by the owner of the reference implementation, not thread
consensus**, recorded in `docs/OWNER_DECISIONS.md` with their rationale and evidence.
Five requirements are new because of them — _Constraint precedence ladder_, _What the limit
floor books_, _Safe state on a degraded safety input_, _Protection timing comes from device
state history_ plus _An unpersisted protection is reported_, and _Decision outcomes are
named and published_ — and seven were made definite. Two carry a caution a reviewer should
see before the detail:

- **D1 (the master stop halts everything) overrode the recommendation put to the owner**,
  which was dispatch-only. Both arguments are in §6; the option that lost is the one that
  keeps a stopped engine diagnosable.
- **D4 (lower number wins conflicts) contradicts the reference binding's own live
  `Controller` contract.** It is a departure from running production behaviour, and the
  binding has to be reconciled — §19.
