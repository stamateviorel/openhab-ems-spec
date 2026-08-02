# Design notes — open questions

This change had no design notes until now. Everything below was opened by the **wave-1
prototype**, not by #3478: wave 1 built a level plane on top of a price series with no
price plane underneath it, and the seam between the two is where these questions sit. Each
names the prototype's own defect id (see `docs/PROTOTYPE_FEEDBACK.md`) and is recorded,
never answered.

## 1. Cost of a load curve versus "the N cheapest slots" (L16)

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

_Time resolution_ mandates non-uniform and mixed intervals inside one series, so candidate
windows covering unequal amounts of time are guaranteed rather than exceptional. Nothing
says how two such windows are compared — by total cost, by cost per unit of energy, by cost
per hour. The prototype's two selection strategies demonstrably disagree once the series
switches to 15-minute entries.

Open: how the shared calculations rank windows of unequal covered time.

## 3. Does a gap break a consecutive window? (L13)

Half of this is already settled here: _Time resolution_ says a series may be non-uniform
and may mix intervals. Contiguity is a different property from uniformity, and nothing says
whether "consecutive" means adjacent entries or an unbroken stretch of covered time — the
two differ exactly when a series has a hole in it, which day-ahead publication makes
routine.

Open: whether a gap breaks contiguity, answered where both consecutive and non-consecutive
selection live.

## 4. Zero and negative consumption prices (L19)

Currency and unit conversion, VAT, fees and taxes are specified here (_Price component
composition_, _Generic grid-price provider_), and negative **feed-in** prices are specified
in _Feed-in pricing_. Negative **consumption** prices are mentioned in no change at all,
though they occur in the same markets that produce the negative feed-in case.

Open: what an effective consumption price of zero or below means to the shared
calculations, and to anything ranking slots on top of them. The visible symptom is
`energy-levels` publishing "blocked = the most expensive slots" on a day whose prices are
all negative.

## 5. Price keys that arrive before the price plane (B3)

A wave-1 declaration mechanism meets keys belonging to this change — a provider's `price` —
before anything can act on them; the prototype recognised such keys and warned. Whether an
unactionable key is accepted and ignored until the capability ships, or rejected, is
`define-extension-points`' question (its §5.3). It is noted here because the keys are this
change's vocabulary, and the answer decides whether a user may declare them early.
