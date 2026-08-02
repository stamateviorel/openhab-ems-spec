# Design notes — open questions (answered 2026-08-02)

This change had no design notes until now. Everything below was opened by the **wave-1
prototype**, not by #3478: the prototype built the runtime limit floor that this change's
planner sits above, and these are the places where the two do not yet meet. Each names the
prototype's own defect id (see `docs/PROTOTYPE_FEEDBACK.md`).

> **Decision pass, 2026-08-02.** The owner answered all five sections; the record, with
> rationale and evidence, is
> [`docs/OWNER_DECISIONS.md`](../../../docs/OWNER_DECISIONS.md). Everything marked
> _Answered_ is an **owner decision in a reference implementation, not thread consensus**.
> No option was deleted — each section keeps what it carried, so overturning a decision
> costs one requirement rewrite.

## 1. Which power figure the budget is denominated in (E4)

_Load balancing under a power budget_ schedules consumers "using their declared maximum
power". No participant class declares one. A Simple consumer carries a surplus
`onThreshold` described as "typically rated power" and nothing else, so the prototype's
budget check had to reuse the threshold as a rating — and a threshold a user set with
margin, in either direction, then makes the budget arithmetic quietly wrong with nothing
signalling that the figure was borrowed.

The missing declaration is being added in `define-participant-model` (E4). What is left
here: which figure this change's budget is denominated in — a nameplate rating, an expected
draw, or a measured one — and whether the runtime floor and the planner may use different
ones.

**Answered — D15 (pack A6) and D16 (pack A14).** _Decision:_ **yes, deliberately
different, and each is stated.** The planner books **declared** figures — Controllable's
declared `max`, Batch's `ratedW` × curve (peak for admission), Simple's optional declared
`ratedPower` falling back to the on-threshold **with the borrowing reported as a
declaration gap, never a rejection**, ModeControllable exempt (§2). The runtime floor books
`max(declared, measured)` for anything already running and the declared figure for anything
not started, re-booked from scratch every cycle with **no carried reserve**. A **capacity
budget** is a third thing again and is denominated in the billed, measured quantity (§3).
`ratedPower` is optional precisely so live Simple declarations stay valid. _Alternatives
preserved:_ reusing the on-threshold as the rating (silently over-books, always in one
direction, because thresholds carry margin) and measure-only (can never reserve room to
start anything) both stand as the readings not taken; so does carrying an error term or
reserve, rejected because it makes the floor's output depend on history.

## 2. Mode changes under a budget (E5)

A ModeControllable consumer's modes carry no power semantics at all, so a budget cannot
tell what "encouraged" draws and nothing can be enforced against a mode change. The
prototype needed an explicit policy for demand it could not quantify, which is a knob for a
gap rather than a choice between framed answers.

Open for the planner (the engine-side half is filed under `define-engine-contract`, E5):
may a mode carry an expected draw, or are mode changes outside the budget?

**Answered — D15 (pack A6).** _Decision:_ **mode changes are outside the planner's budget
by default**, with an **optional per-mode declared draw** that upgrades a mode load into
it. The exemption is grounded rather than lazy: an SG-ready mode 3 draws whatever the heat
pump wants, so a mandatory declared number would be fiction. The cost is stated plainly and
is the same one the reference implementation already names — mode loads' draw is unknown to
the surplus budget. _Alternative preserved:_ requiring every mode to carry an expected draw,
which the decision rejects for SG-ready devices but which the optional form admits for any
device that genuinely knows its numbers.

## 3. Estimates versus measurements over a billing period (E14)

_Peak-based fee models_ is settled in measured, billed quantities — a month's billed
15-minute peak is a meter reading. A planner has to book estimates to allocate capacity
before the fact, and commands are envelopes rather than orders, so the two quantities
diverge within every period. Nothing says which one a budget is denominated in, or how the
error is reconciled as the period runs.

Filed from the engine side as E14; recorded here because the peak-fee archetypes are where
the difference has a monetary consequence rather than a cosmetic one.

**Answered — D16 (pack A14).** _Decision:_ **denominate in the billed, measured quantity —
the meter's own slot average** — and let the planner book against a **projection**: the
running slot average extrapolated to slot end, **reconciled every cycle**, shedding against
`max(month-to-date billed peak, minimum billable demand)` less an explicit, configurable
safety margin. Two halves a naive answer misses, both load-bearing and both in production:
the **minimum billable demand floor** (below it, peaks are free and there is nothing to
shave) and the **month-to-date peak as free budget** (the corpus's own _Established peak as
budget_ scenario already assumed this half). Slots align to the supplier's metering quarters
in **wall-clock zone-local time**, never a rolling window — a rolling 15-minute average
never matches a bill. _Alternatives preserved:_ reconciling at period end (you find out you
blew the month's peak after it is billed) and a feedback loop inside the runtime floor
(which would need a fixed-point argument nobody has, to keep "the same inputs always resolve
one overload the same way" true) both stand as the readings not taken.

_The margin's value_ (D17, Part B row "capacity shed margin", 2026-08-02): the requirement
now ships **300 W**, seeded from the reference's `CAPACITY_SHED_MARGIN_W`, and it is a
**default a site changes**, not a property of the tariff — which is why it sits in the
parameter pass rather than in the decision above it. _Alternative preserved:_ expressing
the margin as a **fraction of the threshold** rather than as watts, which scales with a
site's connection size instead of needing a per-site number; it was not put to the owner,
and it changes what "300" means rather than what the mechanism is.

_Evidence._ This is the mechanism running against a real Belgian capacity-tariff bill in the
reference implementation: a tracker keeping a running average over the current 15-minute
slot "aligned to :00/:15/:30/:45 — same as the supplier's metering quarters", committing at
rollover, tracking the month-to-date peak and resetting monthly, with the shaving controller
comparing the projected quarter against `Math.min(mtdPeak, −minBillableW)` less a shed
margin. Norway's top-3 is the same shape with N-highest averaging.

**Read that expression carefully before copying it, because the corpus deliberately does
not.** In the reference both terms are **negative** — `monthlyPeakW()` is an import figure
under grid-positive-is-export, which is why the same class negates it to display kilowatts
(`mtdPeakKW = -ctx.monthlyPeakW() / 1000.0`) — so `Math.min` there selects the **larger
import magnitude**. Restated in the unsigned watts this requirement uses, that selector is
`max`, and the requirement says `max`. Getting this backwards is not cosmetic: with `min`,
a site whose month-to-date peak is already 4.2 kW would shed everything above the 2.5 kW
minimum billable demand for the rest of the month, contradicting _Established peak as
budget_ two requirements above and costing the user comfort for no billing benefit. The
requirement therefore also states which quantity it is denominated in — an unsigned import
magnitude — so the selector cannot be read without its signs.

## 4. What the boiler vector actually exercises (E16, L18)

`fixtures/expected-boiler-control.csv` is attributed to _Load balancing under a power
budget_. Reproducing it needs look-ahead rescheduling — moving a load to a cheaper hour —
and no wave-1 requirement describes that: a runtime floor can defer a load, it cannot
re-plan one. The prototype reproduced the vector exactly as "the next three cheapest hours,
excluding the heating hours", so the vector cannot discriminate a naive exclusion scheduler
from a real budget scheduler.

Open: does this change own the look-ahead and replanning behaviour the vector implies (the
engine-side half of that question is filed under `define-engine-contract`), and does a
vector that _can_ discriminate the two exist — two loads that could share a slot, or one
that only partially fits? Any such vector has to come from the thread or a production
system; it must not be invented.

The arithmetic is worth stating because it is what makes the vector indiscriminate: the
two loads draw 9 kW and 3 kW against a 10 kW budget, and 9 + 3 = 12 > 10, so under this
budget they can never share a slot and the budget constraint degenerates to mutual
exclusion. Recorded in [`fixtures/README.md`](../../../fixtures/README.md) as well, so a
reader meeting the data first meets the caveat with it.

**Ownership answered — D16 (pack A14).** _Decision:_ **`define-grid-constraints` owns
look-ahead and replanning; `define-engine-contract` owns the runtime floor and a
deferral-to-report path only.** A deferral is **terminal for the cycle** — published as an
outcome on the observability surface (D16 pack A8) — which this change's planner may consume
on its own cadence. So the two changes do not both claim replanning: the floor never
re-plans, and the planner never reaches inside a cycle. `define-engine-contract` carries the
mirror statement; the requirement _Look-ahead and replanning live with the planner_ below is
where this change takes ownership.

**The discriminating vector is answered as sourcing, not as data — D19.** _Decision:_
**derive one from the owner's own site** — four EV chargers sharing an ECO solar budget
under the Belgian capacity tariff is genuinely a partial-fit vector, and a second production
system is the corpus's own accepted provenance class, so this is sourcing rather than
inventing. **It does not exist yet.** No expected schedule has been derived, nothing has
been added to `fixtures/`, and nothing here should be read as though a discriminating vector
were available: the existing pair still cannot tell a budget scheduler from a naive
exclusion scheduler, and passing it stays **necessary but not sufficient**. Recorded as
task 1.3, which is explicit that the numbers come from a real run in site persistence and
must not be fabricated. _Alternatives preserved:_ asking masipila or mstormi in the thread
(best provenance, may never arrive) and shipping without one, keeping the non-discriminating
note (what the corpus does today, and what it still does until task 1.3 lands).

## 5. "Higher-priority" here asserts no direction (E1, A4)

_Load balancing under a power budget_ schedules "using their declared maximum power and
priority", and its scenario reads "the higher-priority load gets the best window". That
sentence reuses the participant model's word without asserting which end of the scale is
better, and the scale's direction, default and tie-break are all still open — they belong
to _Priority_ in `define-participant-model`, which was sharpened to demand that they be
stated (`define-participant-model` design.md §5.2). `define-engine-contract` carries the
same note for the same wording in its own requirement (`define-engine-contract` design.md
§19); this one exists so the reader of a scheduling scenario does not infer a direction
from it either.

The fixtures are the only place a direction is currently observable: this scenario is the
`expected-heating-control.csv` / `expected-boiler-control.csv` pair, where priority 1
heating takes the cheap hours and priority 2 boiler takes what is left — lower = better.
That is evidence of what one worked example did, and it points the opposite way from the
level codes in `expected-planned-levels.csv`, where 3 is the cheapest band. Two ordinal
scales, opposite directions, neither stated in prose (`define-energy-levels` design.md §2,
L1). Recorded, not resolved.

**Answered — D4.** _Decision:_ **lower number = better, in both senses** — served first
_and_ wins conflicts — with **default 100** and ties broken by **participant id ascending**,
which keeps the tie-break stateless. The two ordinal scales stay deliberately opposed:
priority is a **rank** (lower = better), the level scale is a **magnitude** (higher = more
available, fixed 0–3 by D12), and neither is bent to point the same way as the other. The
fixtures therefore keep their meaning — priority 1 heating outranks priority 2 boiler — and
this change's scenario now says so instead of leaving the reader to infer it. The normative
_Priority_ requirement belongs to `define-participant-model`.

**⚠ Known follow-on, recorded not hidden.** The owner's live `emsmanager` binding states the
opposite for conflicts — its `Controller` contract reads "lower runs first, higher wins on
conflict" — so the binding and this corpus now disagree and the binding needs reconciling.
The pack also retracted its own claim that `buildEngineDispatchSet` proved lower-wins; the
evidence that survives is `sortByPriority` (ascending, ties by id, `DEFAULT_PRIORITY = 100`)
and the fixtures. _Alternatives preserved:_ higher = better; split direction (lower serves
first, higher wins conflicts) — the live binding's reading, and the one that would cost the
least to adopt if a maintainer prefers it.
