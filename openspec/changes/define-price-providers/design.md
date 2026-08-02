# Design notes — open questions

This change had no design notes until now. Everything below was opened by the **wave-1
prototype**, not by #3478: wave 1 built a level plane on top of a price series with no
price plane underneath it, and the seam between the two is where these questions sit. Each
names the prototype's own defect id (see `docs/PROTOTYPE_FEEDBACK.md`).

Sections carrying an **ANSWERED** block were decided by the owner on 2026-08-02; the
record, with rationale and evidence, is `docs/OWNER_DECISIONS.md`. Those are **the owner's
decisions in a reference implementation, not thread consensus** — nobody in #3478 agreed
them. The question each one answers is preserved underneath it, including every option that
was not chosen, so overturning a decision costs a rewrite of the named requirement rather
than an excavation. Sections without such a block are still open.

## 1. Cost of a load curve versus "the N cheapest slots" (L16)

> **ANSWERED — owner decision D16, packs A12 and A13** (2026-08-02,
> `docs/OWNER_DECISIONS.md`). Neither reading: **one calculation, `cost(window, weights)`,
> with flat weights as the default**. Slot selection and curve costing are the same function
> called with different weights — a wave-1 caller passes nothing and gets flat selection,
> wave 3 passes a profile curve. The two requirements were never in disagreement; the
> calculation was missing a parameter.
>
> The curve itself belongs to the **profile**, is stored as a relative-time shape and
> becomes a `TimeSeries` only when it is scheduled, anchored by the planner at the candidate
> start; a sample is the average power from its own timestamp to the next, the last interval
> ending at the declared runtime, and the energy integration rule is named: **LEFT Riemann**,
> which is core's own default in `PersistenceExtensions.RiemannType`. That half is pack A13,
> whose home is `add-named-profiles`; it is recorded here because the costing consumes it.
> Stated in _Shared window calculations_.
>
> Alternatives preserved: curve-aware costing exclusively in the shared calculations with
> selection staying curve-blind (which defers a signature change to wave 3), and a curve
> owned by the demand rather than the profile.

`energy-levels` _Window strategies_ selects the N cheapest slots, which minimises cost only
for a load that draws flat. Two requirements already say otherwise: `energy-participants`
_Demand declaration_ asks for a start time that "minimizes cost over the actual curve, not
over an assumed flat draw", and _Shared window calculations_ here already requires "cost of
a given load curve in a given window". The prototype's selection strategies have no access
to a curve — nothing in wave 1 hands them one.

Open: is curve-aware costing exclusively the shared calculations' job, with slot selection
staying curve-blind, or do both consume the same calculation? `add-named-profiles` carries
curves as TimeSeries and inherits the answer.

## 2. Comparing windows that cover unequal time (L14)

> **ANSWERED — owner decision D16, pack A12** (2026-08-02, `docs/OWNER_DECISIONS.md`), by
> removing the question rather than by answering it: **a request carries a `Duration`**, so
> every candidate covers the same amount of time and there is nothing unequal to compare.
> The slot-count entry point kept for the "N cheapest slots" case is the one place the
> question survives, and there candidates are compared on **duration-weighted mean price**.
> Evidence: masipila's production `GenericOptimizer` takes float hours (`allowPeriod(3.25)`)
> and has never had a slot count.
>
> Alternatives preserved: a slot-count API with a stated comparison rule — total cost, cost
> per unit of energy, or cost per hour — which is what the corpus would need if requests ever
> go back to counts.

_Time resolution_ mandates non-uniform and mixed intervals inside one series, so candidate
windows covering unequal amounts of time are guaranteed rather than exceptional. Nothing
says how two such windows are compared — by total cost, by cost per unit of energy, by cost
per hour. The prototype's two selection strategies demonstrably disagree once the series
switches to 15-minute entries.

Open: how the shared calculations rank windows of unequal covered time.

## 3. Does a gap break a consecutive window? (L13)

> **ANSWERED — owner decision D16, pack A12** (2026-08-02, `docs/OWNER_DECISIONS.md`).
> **Contiguity means abutting**: each slot ends exactly where the next begins, so a gap
> breaks a consecutive window. It is one comparison in the implementation, and it keeps
> "consecutive" a statement about covered time rather than about adjacency in a list — which
> is what a non-uniform series needs. Stated in _Shared window calculations_, with a scenario.
>
> The alternative — "consecutive" meaning adjacent entries whatever the hole between them —
> is preserved.

Half of this is already settled here: _Time resolution_ says a series may be non-uniform
and may mix intervals. Contiguity is a different property from uniformity, and nothing says
whether "consecutive" means adjacent entries or an unbroken stretch of covered time — the
two differ exactly when a series has a hole in it, which day-ahead publication makes
routine.

Open: whether a gap breaks contiguity, answered where both consecutive and non-consecutive
selection live.

## 4. Zero and negative consumption prices (L19)

> **ANSWERED — owner decision D17** (Part B row EL-13; 2026-08-02,
> `docs/OWNER_DECISIONS.md`). Prices are **signed and unclamped** throughout the plane —
> `Number:EnergyPrice` is a signed `QuantityType` with no non-negativity constraint, so
> composition carries a negative effective price through and the shared calculations rank the
> signed values unchanged. The visible symptom is answered on the level side rather than
> here: a level is a **rank within the horizon, never a claim about the absolute price**
> (`define-energy-levels` design.md §13), so "blocked" on an all-negative day means "dearest
> of a cheap day" and nothing more. Stated in _Price component composition_ and
> _Shared window calculations_, each with a scenario.
>
> The alternative — an absolute-price condition that overrides the ranking — is preserved;
> the decision routes it through the opt-in escalation seam rather than through the
> calculations.

Currency and unit conversion, VAT, fees and taxes are specified here (_Price component
composition_, _Generic grid-price provider_), and negative **feed-in** prices are specified
in _Feed-in pricing_. Negative **consumption** prices are mentioned in no change at all,
though they occur in the same markets that produce the negative feed-in case.

Open: what an effective consumption price of zero or below means to the shared
calculations, and to anything ranking slots on top of them. The visible symptom is
`energy-levels` publishing "blocked = the most expensive slots" on a day whose prices are
all negative.

## 5. Price keys that arrive before the price plane (B3)

> **STILL OPEN here.** The decision pack dispositioned this one in its Part C — as already
> answered by core's own behaviour rather than by an owner decision — and the owner recorded
> no decision on it, so nothing in this corpus states an answer yet. It remains
> `define-extension-points`' question.

A wave-1 declaration mechanism meets keys belonging to this change — a provider's `price` —
before anything can act on them; the prototype recognised such keys and warned. Whether an
unactionable key is accepted and ignored until the capability ships, or rejected, is
`define-extension-points`' question (its §5.3). It is noted here because the keys are this
change's vocabulary, and the answer decides whether a user may declare them early.
