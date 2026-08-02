# Design notes — open questions

Every question on this page was surfaced by the **wave-1 prototype**, not by the #3478
thread (see docs/PROTOTYPE_FEEDBACK.md); the defect id is given in each heading so a
reviewer can tell prototype-driven repair from thread-sourced consensus. Thread and
production sources are cited inline where one exists. Nothing here is answered — this
page exists because this was the only wave-1 change without a `design.md`, so its open
questions lived in `tasks.md`, where the prototype rules never point a reader (N8).

## 1. Level names — the spec and the taxonomy it cites disagree (A11 / L11)

The requirement names the levels `blocked / normal / encouraged / overcapacity`; the
storm.house taxonomy it cites in the same Source line names them
`restricted / normal / encouraged / maximum`. Two of four differ. The prototype used the
spec's names and flagged the mismatch rather than reconcile it.

Which vocabulary is normative is a maintainer call — **for @mstormi (whose model the
names come from) and @masipila (whose SG framing they were matched to)**. It is task 1.1.
Note the user-facing consequence: whichever set wins ends up in Item state options, UI
labels and every translation file, so a later rename is not cheap.

## 2. Ordinal encoding and its direction (L1)

The prose orders the four levels but never numbers them. The evidence in the corpus
today, which the requirement now demands be stated in prose instead:

- `fixtures/expected-planned-levels.csv` encodes the **cheapest** slots as `3` and the
  **most expensive** as `0`. Because `docs/PROTOTYPE_TRACK.md` makes fixtures pass/fail,
  that file already binds every implementation — without any requirement saying so.
- `energy-participants` _Priority_ orders consumers with no stated direction either, and
  the prototype adopted lower = better there (A4). Wave 1 therefore ships two ordinal
  scales that, on the only readings the corpus supplies, point **opposite** ways.

Open, in this order: is a numeric encoding fixed centrally at all? The requirement is
worded so that this stays askable — it demands a stated encoding _wherever a level is
exchanged as a number_, which a third option satisfies without any canonical value: the
scale is named, core states no numbering, and each mapping (SG-ready 1–4, an Item's state
options, a fixture) is stated by whoever exchanges it. That option is not free — §1 notes
the level names already end up in Item state options, UI labels and every translation
file — but it is live, and neither the requirement nor this note rules it out. If an
encoding _is_ fixed centrally: does the fixture's become the normative one, or is the
prose free to choose another and the fixture regenerated? And should the two ordinal
scales in wave 1 be made to point the same way, or is "level = more availability is
higher, priority = better is lower" an acceptable asymmetry a user never sees? None of the
three is decided here; recording the fixture's de-facto encoding is evidence, not a
verdict.

## 3. SG-ready: the offset, other mode counts, and mode 1's time cap (L2, A6)

_Four-level scale_ promises an SG-ready mapping "without translation logic in user
rules". SG-ready numbers its modes **1–4**; the level codes the fixture binds run
**0–3**. An offset therefore exists and the requirement never pins it, so a reader who
saw only the fixture would publish 0–3 straight onto an SG-ready channel.

Two further parts, both open:

- The mapping is specified for four modes only. How does a four-level site signal map
  onto _n_ ordered modes for _n_ ≠ 4 — heat pumps with two or three inputs, EVCC-style
  controllers with a different set (A6, `energy-participants` _Four consumer profile
  classes_)?
- SG-ready mode 1 (the blocking mode) is widely documented as carrying a cap on how long
  per day it may be asserted — the prototype reported it as a legal cap, and no primary
  source for it is cited anywhere in this corpus, so **it needs confirming against the
  SG-Ready specification before anything is built on it**. That cap appears nowhere in the
  corpus either way. If it is real: is honouring it in scope for core, a participant-side
  concern, or explicitly out of scope?

## 4. One level, or one level per domain (I4, task 1.2)

Task 1.2 leaves site-global versus site + per-domain open. The prototype found that this
is the first place the undecided task becomes a **signature** rather than a preference: a
site-global answer makes the level source a getter, a per-domain answer makes it a keyed
lookup, and the engine snapshot carries exactly one level either way. It kept the seam
single-valued and flagged it, having come within one review of settling it silently.

The framing belongs here, not only in `tasks.md`. The task stays unanswered.

## 5. Derivation: what the prototype proved task 2.1 is actually about (L12)

Task 2.1 asked whether derivation is percentile-based (the emsmanager reference) or
fixed-hour-count (storm.house). The prototype ran both over `fixtures/dayahead-prices.csv`
and they produced **identical** output: percentiles at 1/6 · 1/6 · 1/6 reproduce
`expected-planned-levels.csv` exactly, as do fixed counts of 4/4/4. On tie-free data the
two formulations are the same function.

They diverge only where prices repeat: a percentile threshold is a price and cannot split
a set of equal prices, while a count cuts the ranking wherever it lands. So the open
content of task 2.1 is **tie handling** (see §6) plus **adaptivity** — whether the bands
follow the day's price spread or stay a fixed size — not "which algorithm". Task 2.1 has
been reworded to that question. It is still unanswered, and this note picks neither.

## 6. Tie-break, band precedence and counts that do not fit (L3, L5, L6)

_Level derivation_ now requires a stated tie-break; **which** rule it is remains open.
`fixtures/README.md` already says one is needed and that the corpus dataset happens to
contain none; real markets repeat prices constantly (zero and negative hours), so this is
routine, not a corner. The prototype ranked equal prices by earlier slot start, in one
place, so a maintainer decision changes one comparator.

Two neighbouring silences, both open:

- **Counts that overflow the series.** Day-ahead publishes around 13:00, so a live series
  is routinely 11 or 35 slots rather than 24, and the configured counts may not fit. What
  happens then — clamp (and in which band order), scale, or refuse to plan? The prototype
  clamped, cheap bands first.
- **Band precedence even without overflow.** The three bands are described independently
  over one ranking; only `sum ≤ n` keeps them disjoint. Which band wins where they
  overlap is unspecified.

## 7. Hours or slots (L4)

_Level derivation_ configures "4 overcapacity, 4 low-price and 4 blocked **hours**";
_Window strategies_ requires independence from 60- and 15-minute resolution, and
`price-data` _Time resolution_ mandates non-uniform and mixed intervals inside one
series. On 15-minute data "4 hours" is either 4 slots or 16 — the two requirements in
this one change disagree on the unit.

Open: are the configured counts expressed in **hours** (a duration, needing a rule for
partial slots) or in **slots** (a count, needing nothing but changing meaning with
resolution)? The prototype counted slots and supplied a converter that _refuses_ a
mixed-width or non-whole-multiple series rather than invent a rounding rule. **The unit
is a decision; it is not taken here.**

## 8. Escalation has no magnitude, and no hysteresis (L7, L8)

_Level derivation_ says the level "escalates" on live PV excess. Nothing states the
**threshold** at which it escalates, the **magnitude** (one step, or straight to
overcapacity), or the **target level**. Both readings are present in the sources, so the
requirement is inert until someone configures it; the prototype shipped `none`,
`maximum` and `graded` as configuration and defaulted to the already-observable
behaviour rather than to a verdict.

Separately, "for the duration of the surplus" describes a stateless function of the
instantaneous reading. On a partly cloudy day that makes the published site level chatter
minute to minute, which every production system solves with a minimum dwell time or a
deadband. Open: does the current level carry hysteresis, and is that a core guarantee or
per-site configuration?

Both questions sit on top of a third that this change does not own: **the corpus nowhere
defines what surplus is** (E7, filed under `define-engine-contract` design.md §11).
Whether a charging battery's power counts, whether curtailed PV counts, whether an
already-running managed load's own draw counts, and whether the figure is instantaneous or
averaged, each move the escalation trigger — so a threshold and a dwell time cannot be
argued about usefully until the quantity they apply to is named. It is also the reason
_Level derivation_'s "PV escalation" scenario sits inside the I1 recursion (§15): the
surplus is part of the engine's snapshot, and the level is a function of the surplus.
`define-optimization-objectives` design.md §6 records the same dependency for its
self-consumption objective, which places a load "into the surplus".

## 9. The current level outside the plan (L9)

_Planned schedule vs. current level_ says live conditions may override the plan for the
current slot, but never says what the current level is when there **is** no plan for the
current slot: before the first prices arrive, after the plan's last slot, or inside a gap
in a non-uniform series. Candidate answers: the middle level (`normal`) as a neutral
fallback; the most restrictive level, so an unplanned site does nothing; hold the last
planned level until a new plan arrives; or publish no level at all and let every consumer's
gate decide what an absent level means. The prototype made it a configuration parameter
defaulting to `normal`, on the reasoning that it was the least surprising value it could
name — a build artefact, not a proposal, and it is the option that most quietly hides a
missing price feed. The value the corpus intends is undecided.

## 10. When rule-updated counts take effect (L22)

The hour counts are "user-configurable, including via rules" — masipila's rule-updatable
counts and mstormi's cold-weather example both depend on it. Nothing says **when** a
change takes effect: immediately (re-deriving the remainder of today), at the next
midnight boundary, or on the next price arrival. The answer interacts with task 2.2's
re-planning semantics (fresh overwrites old) and should be settled with it.

## 11. Window strategies leave five things undefined (L13, L14, L15, L16, L17)

_Window strategies_ requires consecutive and non-consecutive selection, resolution-
independent. Five silences the prototype had to fill by guessing:

- **Does a gap break contiguity?** Nothing says whether a hole in the series interrupts a
  "consecutive" window. (Half of this is already settled elsewhere and is _not_ open:
  `price-data` _Time resolution_ mandates that a series may be non-uniform and may mix
  intervals, so a classifier may not assume a fixed slot width.)
- **Comparing windows of unequal covered time.** With mixed intervals mandated, two
  candidate windows of the same slot count can cover different amounts of time; which is
  "cheaper" is undefined. Total cost, cost per hour, or only compare equal-duration
  windows?
- **Window start granularity.** May a window start only on a slot boundary, or anywhere?
  The prototype started on slot boundaries.
- **The load's own shape.** "Cheapest N slots" is correct only for a flat load, yet
  `energy-participants` _Demand declaration_ demands a start time that "minimizes cost
  over the actual curve, not over an assumed flat draw", and nothing in _Window
  strategies_ references the curve. Note this is partly a wave question:
  `price-data` _Shared window calculations_ already requires "cost of a given load curve
  in a given window" in wave 2. Open: does wave-1 slot selection account for a declared
  load curve, or is curve-aware selection deferred to the shared calculations — leaving
  the two requirements in tension until then?
- **Under-supply.** With fewer eligible slots than requested, is the answer a partial
  selection or nothing? It is user-visible, and it plausibly differs between the two
  selection kinds — the prototype returned a partial set for non-consecutive requests and
  nothing for consecutive ones, and both were guesses.

## 12. Season grammar: boundaries, zone, and which date names a day (L20 / L21 / I3)

_Seasonal window defaults_ makes seasonal parameter sets "selectable automatically by
date" and defines no grammar for a season. Three things are missing and each blocks a
configuration schema:

- **Boundaries.** How is a season expressed — fixed month-day pairs, meteorological or
  astronomical seasons, an arbitrary user-defined set of date ranges?
- **Time zone.** Whose midnight starts a season: the site's zone, the market's, or UTC?
- **The naming date of a delivery day that straddles two.** The corpus's own fixture
  starts at 23:00 UTC the evening before the day it describes, so this is not
  hypothetical: a delivery day routinely spans two local dates and nothing says which one
  names it.

Because a flat configuration map cannot express any of this without inventing a grammar,
the prototype made seasonal derivation reachable only programmatically and offered no
configuration value for it.

## 13. What the levels mean when prices are zero or negative (L19)

"Blocked = the most expensive slots" is a strange thing to publish on a day when every
consumption price is negative — being paid to consume, and told not to. Ordering-only
code survives it; the published meaning does not.

Scope note, so this is not re-raised wholesale: currency, unit conversion, VAT, fixed
fees and conditional tariffs are already specified in `price-data`
(_Prices as future-timestamped series_, _Price component composition_,
_Generic grid-price provider_), and negative **feed-in** prices in _Feed-in pricing_.
The residue is genuinely uncovered: negative and zero **consumption** prices are never
mentioned anywhere in the corpus. Open: do the four levels stay purely relative (rank
within the day, whatever the sign), or does an absolute-price condition override the
ranking?

## 14. One Item or two, and what publication means under shadow (L23, I2)

The requirement calls the plan "a future-timestamped TimeSeries" and the current level
"an Item". openHAB's `TimeSeries` is per-item, so whether the plan rides on the **same**
Item as the current level or on a second one is unstated — a difference a user sees in
their item list and their charts.

Underneath it sits a bigger unanswered one, cross-cutting with `define-engine-contract`
(I2): the level plane is specified as **publishing** Items while the engine **consumes**
a level. Under shadow-only nothing may be written, so publication cannot exist yet —
which means the only classifier→engine path is in-process, and the corpus describes
none. The prototype added an in-process seam. If the maintainers intend the engine to
read the published Item instead, the classifier and the engine are coupled through the
item registry and that seam disappears: a materially different architecture, and one the
specification should state outright rather than leave to an implementer. The same
question belongs in `define-engine-contract`'s design notes.

## 15. Is the site level an input to the engine, or a product of it (I1)

_Level derivation_'s "PV escalation" makes the current level a function of live surplus.
`define-engine-contract`'s _Central periodic evaluation_ makes the surplus part of the
one consistent snapshot each cycle reasons about. As written the two are mutually
recursive, and neither change mentions the other.

Composed naively the engine reads the grid twice in one cycle — once for the level, once
for the snapshot — and the two readings can disagree inside that cycle. The prototype
removed that inconsistency by resolving the level from the readings the snapshot already
took, and reports it as the removal of a contradiction rather than an answer, because the
corpus never says **which way the dependency runs**: is the site level an input handed to
the engine, or a product of the engine's own cycle?

This one must be answered in both places at once. The same question belongs in
`define-engine-contract`'s design notes (§4, context snapshot content).

## 16. Provenance

Nothing on this page is thread consensus. Every section is a silence, contradiction or
inertness the wave-1 prototype hit while building this change, recorded so that a reader
of `tasks.md` is not the only person who learns the questions exist. Where a thread or
production source bears on a question it is cited inline; where the prototype guessed to
compile, its guess is stated as a guess so a maintainer can see what a decision would
cost. Sections 14 and 15 are cross-cutting and are recorded on both sides — see
`define-engine-contract/design.md` §18 (I2, how the site level reaches the engine) and §4
(I1, input or product).
