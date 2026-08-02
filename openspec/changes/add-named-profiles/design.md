# Design notes — open questions (answered 2026-08-02)

This change had no design notes until now. Everything below was opened by the **wave-1
prototype**, not by #3478: wave 1 had to model a load curve before this change makes one a
named, stored object, and the modelling ran out of specification in three places. Each
names the prototype's own defect id (see `docs/PROTOTYPE_FEEDBACK.md`).

> **Decision pass, 2026-08-02.** The owner answered all three sections in one ruling (D16,
> pack A13) and added a fourth (D13); the record is
> [`docs/OWNER_DECISIONS.md`](../../../docs/OWNER_DECISIONS.md). These are **owner
> decisions in a reference implementation, not thread consensus**. Nothing was deleted —
> the readings each section carried are left standing, so overturning one costs a
> requirement rewrite rather than a reconstruction.

## 1. Is the curve the demand's, the programme's, or the profile's? (A7)

`energy-participants` _Demand declaration_ describes the load curve as part of the
**demand** ("Batch-class demand additionally carrying a load curve"); the four-classes
taxonomy in the same change treats it as a property of the **programme**. Both readings are
in the text, and the prototype had to pick one to compile. This change introduces a third
position — the curve belongs to a named **profile** — which is a defensible reconciliation
and also a statement that has to line up with whatever `define-participant-model` answers.

Open: where the curve lives, and whether _Profile contents_ here supersedes the demand-side
reading or coexists with it.

**Answered — D16 (pack A13).** _Decision:_ **the curve belongs to the profile**, and this
change's reading supersedes the demand-side one. A consumer always has exactly **one active
profile** — implicit and unnamed in wave 1, named here in wave 3 — so the two waves are one
model rather than two, and wave 1's `ratedW × runtimeHours` is exactly the rectangular
special case of a profile curve. _Why:_ curve-on-demand means a Batch consumer with no
pending demand has no curve, so nothing can pre-cost a programme before the user asks; and
two demands on one dishwasher could declare different curves for one physical programme.
_Alternatives preserved:_ curve-on-the-demand and curve-on-the-programme both stand above
as the readings not taken; the demand-side wording in `define-participant-model` is the
place a maintainer would reverse this.

## 2. What a sample means, and how many there may be (A8)

Nothing anywhere bounds a curve's length, requires a sample spacing, or says whether a
sample is instantaneous power or the average over its interval. _Profile contents_ raises
the stakes rather than settling them: it explicitly wants a white-goods profile at 1-minute
spacing over a day and a heat-pump profile with mixed intervals over a year in one
representation, so the answer decides both the storage cost of a profile and the meaning of
the scale factor applied to it.

Prototype evidence: its curve type is evenly spaced over the declared runtime, with values
required to be finite and non-negative, no upper bound, and no stated sample semantics.
Every one of those is invention filling a silence, not a reading of the corpus.

**Answered — D16 (pack A13).** _Decision:_ **a sample is the average power over the
interval from its own timestamp to the next sample's**, the last interval ending at the
declared runtime, with **spacing carried per sample rather than implied**. The energy
integration rule is stated by name: **LEFT Riemann** — reusing core's own term, since
`PersistenceExtensions.RiemannType` already defines `LEFT, MIDPOINT, RIGHT, TRAPEZOIDAL`
with LEFT as core's default. Sample count is capped only as a **declaration guard** (say
1440), carrying no semantic meaning. _Why:_ leaving sample semantics unstated means two
conforming implementations cost the same dishwasher differently. _Alternatives preserved:_
instantaneous-power samples, evenly-spaced-only curves (the prototype's invention) and
MIDPOINT/RIGHT/TRAPEZOIDAL integration all remain available to a maintainer who prefers
them; each changes the cost of a shaped window, not the shape of the model.

## 3. A stored series versus an anchored shape (no defect id)

The prototype's curve is deliberately relative-time — a shape anchored only when it is
scheduled — and deliberately not a TimeSeries. _Profile contents_ here mandates a
TimeSeries, which is absolute-time. Both can be true if a stored profile is an absolute
series that is shifted onto the plan, but nothing says where that anchoring happens or what
the timestamps of a stored profile mean before it is scheduled.

Open: relative shape or anchored series, and what a profile's timestamps denote at rest.
The answer decides whether a profile can be carried through the same storage plane as
prices and forecasts, which is the stated reason for choosing TimeSeries in the first
place.

Unlike §1 and §2 this section carries **no defect id**: the same build surfaced it, but the
prototype's own lists do not record it, so it appears in `docs/PROTOTYPE_FEEDBACK.md`
without one. It is prototype-driven all the same.

**Answered — D16 (pack A13), and this one changes a requirement rather than settling a
silence.** _Decision:_ the stored profile is a **relative-time shape** — offset-from-start
plus a fraction of a scale factor — which **becomes a `TimeSeries` at the moment it is
scheduled**, anchored by the planner at the candidate start. A stored profile's timestamps
therefore denote nothing absolute at rest, because it has none: it carries offsets.
_Why:_ core's `TimeSeries` is `(Instant, State)` pairs with no width and no relative mode,
so storing an unanchored curve as one forces its timestamps to denote a fictitious epoch and
makes `Policy.REPLACE` meaningless. _The cost, stated plainly:_ a relative-time profile does
**not** ride the price/forecast storage plane, which was the stated attraction of TimeSeries
in _Profile contents_. It becomes a small configuration entity instead — which task 1.1
already anticipates. _Alternative preserved:_ storing the profile as an absolute
`TimeSeries` and shifting it onto the plan, which is what the requirement said before this
pass; restoring it means rewriting _Profile contents_ and accepting the fictitious-epoch
timestamps.

## 4. Can a named profile lift a hands-off flag? (new — D13)

Not a prototype defect id: the question only exists once `never` becomes a property of the
consumer. Under D13 the hands-off flag lifts **off the Simple class onto the consumer**, so
it applies to all four classes — and a named profile is "a complete parameterization of the
participant's class", which raises the obvious question of whether switching profiles could
switch hands-off off.

**Answered — D13.** _Decision:_ **no.** `never` belongs to the consumer, sits in the
engine-owned prohibition set, and outranks profile selection: no named profile can carry,
clear or weaken it. What a profile _may_ still override is the user-owned parameterization
the requirement already names — demand, deadline, protections — because a profile is the
user's own configuration, and D13's boundary constrains **contributed algorithms**, not the
user. _Why:_ otherwise the devices most in need of hands-off (an EV driven manually, a
dishwasher) get it back by a seasonal profile switch nobody was thinking about, and users
fake hands-off by deleting the declaration, which also deletes the measurement the limit
floor needs. _Alternative preserved:_ making `never` a per-profile parameter, which would
let a site express "hands off in summer" directly — a real use case, at the cost of a
prohibition that can be switched off by a schedule.
