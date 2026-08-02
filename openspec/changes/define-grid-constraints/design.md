# Design notes — open questions

This change had no design notes until now. Everything below was opened by the **wave-1
prototype**, not by #3478: the prototype built the runtime limit floor that this change's
planner sits above, and these are the places where the two do not yet meet. Each names the
prototype's own defect id (see `docs/PROTOTYPE_FEEDBACK.md`) and is recorded, never
answered.

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

## 2. Mode changes under a budget (E5)

A ModeControllable consumer's modes carry no power semantics at all, so a budget cannot
tell what "encouraged" draws and nothing can be enforced against a mode change. The
prototype needed an explicit policy for demand it could not quantify, which is a knob for a
gap rather than a choice between framed answers.

Open for the planner (the engine-side half is filed under `define-engine-contract`, E5):
may a mode carry an expected draw, or are mode changes outside the budget?

## 3. Estimates versus measurements over a billing period (E14)

_Peak-based fee models_ is settled in measured, billed quantities — a month's billed
15-minute peak is a meter reading. A planner has to book estimates to allocate capacity
before the fact, and commands are envelopes rather than orders, so the two quantities
diverge within every period. Nothing says which one a budget is denominated in, or how the
error is reconciled as the period runs.

Filed from the engine side as E14; recorded here because the peak-fee archetypes are where
the difference has a monetary consequence rather than a cosmetic one.

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
