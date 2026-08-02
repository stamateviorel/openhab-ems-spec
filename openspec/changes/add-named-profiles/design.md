# Design notes — open questions

This change had no design notes until now. Everything below was opened by the **wave-1
prototype**, not by #3478: wave 1 had to model a load curve before this change makes one a
named, stored object, and the modelling ran out of specification in three places. Each
names the prototype's own defect id (see `docs/PROTOTYPE_FEEDBACK.md`) and is recorded,
never answered.

## 1. Is the curve the demand's, the programme's, or the profile's? (A7)

`energy-participants` _Demand declaration_ describes the load curve as part of the
**demand** ("Batch-class demand additionally carrying a load curve"); the four-classes
taxonomy in the same change treats it as a property of the **programme**. Both readings are
in the text, and the prototype had to pick one to compile. This change introduces a third
position — the curve belongs to a named **profile** — which is a defensible reconciliation
and also a statement that has to line up with whatever `define-participant-model` answers.

Open: where the curve lives, and whether _Profile contents_ here supersedes the demand-side
reading or coexists with it.

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
