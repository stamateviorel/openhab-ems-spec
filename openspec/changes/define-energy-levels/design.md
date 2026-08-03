# Design notes — open questions

Every question on this page was surfaced by the **wave-1 prototype**, not by the #3478
thread (see docs/PROTOTYPE_FEEDBACK.md); the defect id is given in each heading so a
reviewer can tell prototype-driven repair from thread-sourced consensus. Thread and
production sources are cited inline where one exists. This page exists because this was
the only wave-1 change without a `design.md`, so its open questions lived in `tasks.md`,
where the prototype rules never point a reader (N8).

Sections carrying an **ANSWERED** block were decided by the owner on 2026-08-02; the
record, with rationale and evidence, is `docs/OWNER_DECISIONS.md`. Those are **the owner's
decisions in a reference implementation, not thread consensus** — nobody in #3478 agreed
them. The question each one answers is preserved underneath it, including every option
that was not chosen, so overturning a decision costs a rewrite of the named requirement
rather than an excavation. Sections without such a block are still open.

## 1. Level names — the spec and the taxonomy it cites disagree (A11 / L11)

> **STILL OPEN.** The decision pack recommended renaming to
> `restricted / normal / encouraged / maximum` (its pack A11). The owner took only the
> **encoding** half of that pack (D12, §2) and left the names alone, so this section is
> unchanged and the recommendation above is simply one more live option.

The requirement names the levels `blocked / normal / encouraged / overcapacity`; the
storm.house taxonomy it cites in the same Source line names them
`restricted / normal / encouraged / maximum`. Two of four differ. The prototype used the
spec's names and flagged the mismatch rather than reconcile it.

Which vocabulary is normative is a maintainer call — **for @mstormi (whose model the
names come from) and @masipila (whose SG framing they were matched to)**. It is task 1.1.
Note the user-facing consequence: whichever set wins ends up in Item state options, UI
labels and every translation file, so a later rename is not cheap.

## 2. Ordinal encoding and its direction (L1)

> **ANSWERED — owner decision D12** (2026-08-02, `docs/OWNER_DECISIONS.md`). A numeric
> encoding **is** fixed centrally, and it is the one the corpus fixture already binds:
> `blocked = 0`, `normal = 1`, `encouraged = 2`, `overcapacity = 3`. Rationale: every
> component then round-trips identically and nobody has to guess whether level 3 means SG
> mode 3 or 4. Evidence: `fixtures/expected-planned-levels.csv` encodes exactly this, so no
> fixture is rewritten, and the _Numeric exchange_ scenario forecloses the third option
> (name-only, encoding per exchange) by requiring a round trip between two independently
> written components. Stated in _Four-level scale_.
>
> The direction question is answered too, and the answer is that the two wave-1 ordinal
> scales are **deliberately not** made to point the same way — by **D4**, whose home is
> `define-participant-model`. A level is a magnitude, so a higher number means more
> available energy; a priority is a rank, so a lower number means better. A user sees one
> of them at a time.
>
> The third option below — core states no numbering, and each exchange states its own —
> and the "regenerate the fixture from a different central encoding" variant are preserved
> as written.

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

> **ANSWERED — owner decisions D12 and D17** (2026-08-02, `docs/OWNER_DECISIONS.md`), in
> three parts, and one gap stays open.
>
> _The offset_ (D12): it is exactly one, everywhere — SG-ready modes are 1–4, level codes
> are 0–3. The corpus states the correspondence as a **named mapping** (blocked ↔ mode 1 …
> overcapacity ↔ mode 4) and never as arithmetic on the code, which is also core's own
> convention for pairing a named mode with a level. Stated in _SG-ready mode mapping_.
>
> _Mode counts other than four_ (D17, Part B row PM-5.10): `modeIndex =
> ceil(level × (n − 1) / 3)`, index 0 being the most restricted, and **not** for _n_ = 2 —
> a binary device uses the per-consumer "on at level ≥ N" gate instead, because the formula
> would otherwise place it in its permissive mode at every level above blocked and so
> impose a threshold its owner never declared. (`ceil` rather than `floor` is what keeps a
> **normal** level from dropping a few-mode device into its most restricted mode; that is
> the reason for the rounding, not a reason to apply the formula to _n_ = 2.) That mapping
> belongs to `define-participant-model`, which owns the consumer classes, and is now its
> _Level-to-mode mapping for any mode count_ requirement; it is recorded here because the
> question was asked here. Note for anyone reading the two together: `modeIndex` is a
> zero-based position in the consumer's own ordered list, and for _n_ = 4 it happens to
> equal the level code — turning it into an SG-ready mode **number** is still the named
> correspondence above, never arithmetic.
>
> _Mode 1's time cap_ (D17, Part B row EL-3c): real and quantified — the decision pack
> fact-checked the BWP SG-Ready specification as at most **two hours per assertion** and at
> most **three assertions per day** — and **participant-side, explicitly out of scope for
> core**. The honest gap the decision does not close: the participant model can express
> "at most two hours" as a maximum-off time and **cannot** express "three times per day",
> and this corpus still cites no primary SG-Ready document of its own, so the figures rest
> on that fact-check.

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

> **ANSWERED — owner decision D17** (Part B row EL-4; 2026-08-02,
> `docs/OWNER_DECISIONS.md`). **Site-global**, with the seam shaped as `level()` so that a
> later `level(domain)` can default to it — which is exactly the single-valued seam the
> prototype kept. Evidence: all three production systems publish exactly one level, the
> corpus has no domain model to key on, and the use case that motivated per-domain is
> served per-consumer by the level gate. Reversible by construction: it is the seam's
> signature, and widening a getter is additive.
>
> The per-domain option below is preserved; adopting it changes one signature and the
> engine snapshot's level field.

Task 1.2 leaves site-global versus site + per-domain open. The prototype found that this
is the first place the undecided task becomes a **signature** rather than a preference: a
site-global answer makes the level source a getter, a per-domain answer makes it a keyed
lookup, and the engine snapshot carries exactly one level either way. It kept the seam
single-valued and flagged it, having come within one review of settling it silently.

The framing belongs here, not only in `tasks.md`. The task stays unanswered.

## 5. Derivation: what the prototype proved task 2.1 is actually about (L12)

> **STILL OPEN.** The decision pack proposed retiring this section and §6 together — bands
> configured as a fraction of the series, resolved to an integer slot count, cut from one
> ascending `(price, slot start)` ranking (its pack A11). The owner took only the encoding
> half of that pack (D12, §2), so adaptivity is undecided and the pack's proposal is one
> live option among the ones below.

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

> **STILL OPEN**, and deliberately so. The decision pack would have retired all three
> silences at once (price ascending then slot start ascending as the tie-break; bands cut
> from one ranking, which makes overflow and band precedence structurally impossible — its
> pack A11). The owner took only the encoding half of that pack (D12, §2). Nothing below is
> settled, and the requirement still only demands _that_ a tie-break be stated.

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

> **NARROWED, not answered.** Pack A12 (owner decision D16) fixes the unit for **window
> requests** — they carry a `Duration` — so _Window strategies_ and `price-data` no longer
> disagree with each other about a window. The **band counts** in _Level derivation_ are
> untouched: they are still configured in "hours" against a series that may be 15-minute or
> mixed, and that unit is still a decision nobody has taken.

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

> **ANSWERED in part — owner decisions D17, D10 and D11** (2026-08-02,
> `docs/OWNER_DECISIONS.md`), and part of it is explicitly still open.
>
> _Magnitude and thresholds_ (D17, Part B row EL-8a): escalation is **graded** —
> `encouragedFrom` is a site surplus threshold in watts, `overcapacityFrom` defaults to
> twice it. `[T]` storm.house escalates on a configured watt threshold; the second step is
> the reference implementation's addition. This is what makes the requirement **expressible**
> — the prototype shipped `none` and had no shape to configure at all — and it is active once
> a site sets a threshold.
>
> _An earlier draft of this note claimed the decision made the requirement "stop being inert
> out of the box". It does not, and the correction matters more than the wording._ EL-8a
> fixes the **shape** and the 2× relation between the two thresholds; it never gave
> `encouragedFrom` a base number, and no source in this corpus supplies one. The reference
> implementation has no site-level escalation threshold to read one off — its 2000 W
> `SURPLUS_DEFAULT_ON_THRESHOLD_W` is a **per-consumer** switching parameter for a Simple
> load, a different quantity, and the 1500 W in the requirement's own scenario is a GIVEN
> chosen to make the arithmetic legible, not a sourced default. So the requirement states
> what an **unset** threshold means — no escalation above the planned level — which is
> definite, testable and honest, rather than escalating at a number nobody chose. Supplying a
> default later is a one-line change and needs only a source.
>
> _The quantity the thresholds apply to_ (D10): **surplus = grid export + battery charging
> the engine can reclaim**, because battery charging is a decision the EMS itself made and
> is therefore available to a better-priority consumer (a lower priority number, D4) —
> grounded in the owner's own site,
> where solar-first EV charging that ignored battery charging idled while the battery
> absorbed everything. Signs come with it (D11): grid + = export, battery + = charging,
> PV + = producing, consumers + = consuming, devices that disagree normalised at the edge.
> Alternatives preserved where the question lives: `define-engine-contract` design.md §11
> (grid-export-only, which is what the prototype used; export + battery + curtailed PV) and
> `define-participant-model` design.md §5.11 (per-participant declared sign conventions).
> Both requirements above are stated in _Surplus escalation of the current level_.
>
> _The composition, amended_ (D27, 2026-08-03): surplus is **`max(0, grid + reclaimable)`**,
> not the literal sum of two non-negative terms. D10 as worded over-states it whenever the
> site imports while the battery charges — grid −1 kW with a battery at +3 kW reports 3 kW,
> where stopping the battery frees 2 kW before the meter goes positive again — so a consumer
> started on that figure puts the site straight back into import. **This amends D10 as
> literally worded**, and the owner said so rather than reinterpreting it. Note for whoever
> writes the vectors: the scenario above (_Battery charging is part of the surplus_) has grid
> at exactly 0 and therefore does **not** discriminate between the two readings, which is why
> the requirement now carries an importing-while-charging scenario and a never-negative one.
> _Alternatives preserved_: the literal sum, and publishing an export figure and a
> reclaimable figure separately — both argued in full in
> `define-engine-contract` design.md §11.
>
> _Still open, by the owner's own note on D10_: whether the surplus figure is
> instantaneous, averaged or forecast, and whether an already-running managed consumer's
> own draw counts towards it. The requirement is worded so that neither is foreclosed.
>
> _Hysteresis_ is **not an owner decision**: the decision pack dispositioned it in its
> Part C as already answered by the corpus — chatter is absorbed downstream by each
> consumer's minimum on/off and maximum off times, by requirement, and a level-side
> deadband, if ever wanted, is two thresholds (engage high, release low) and not a dwell
> timer. No requirement was changed for it here, so the paragraph below stands as the
> record of the question.

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

> **ANSWERED — owner decision D17** (Part B row EL-9; 2026-08-02,
> `docs/OWNER_DECISIONS.md`). **`normal`**, with the absence of the plan reported
> separately — which is what removes the objection the prototype itself raised against
> `normal`, that it quietly hides a missing price feed. Evidence: the reference
> implementation's `pricePercentileLevel` returns the normal level for a missing schedule
> and production runs on it; the most-restrictive alternative turns a failed HTTP fetch into
> a cold house. Stated in _Level when no plan covers the present instant_.
>
> The three alternatives below — most-restrictive, hold-the-last-planned-level, publish no
> level at all — are preserved; each is a one-line change to the same fallback.

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

> **ANSWERED — owner decision D17** (Part B row EL-10; 2026-08-02,
> `docs/OWNER_DECISIONS.md`). **Immediately**: re-derive over the whole published series,
> re-publish `[now, end)` as one TimeSeries with `Policy.REPLACE`, leave elapsed entries
> alone. Derivation is a pure function re-run every tick, and core's `Policy.REPLACE`
> already deletes exactly the published range — with the caveat the decision pack asked to
> be carried into the requirement: `PersistenceManagerImpl` converts those bounds using the
> system default zone. Stated in _Re-derivation when the configuration changes_, which also
> settles task 2.2's re-planning semantics.
>
> The midnight-boundary and next-price-arrival alternatives below are preserved.

The hour counts are "user-configurable, including via rules" — masipila's rule-updatable
counts and mstormi's cold-weather example both depend on it. Nothing says **when** a
change takes effect: immediately (re-deriving the remainder of today), at the next
midnight boundary, or on the next price arrival. The answer interacts with task 2.2's
re-planning semantics (fresh overwrites old) and should be settled with it.

## 11. Window strategies leave five things undefined (L13, L14, L15, L16, L17)

> **ANSWERED — owner decision D16, pack A12** (2026-08-02, `docs/OWNER_DECISIONS.md`), all
> five, and the fifth by removing the question rather than by picking a side:
>
> - _Gap and contiguity_: **contiguity means abutting** — each slot ends exactly where the
>   next begins, so a gap breaks a consecutive window.
> - _Unequal covered time_: **requests carry a `Duration`**, so every candidate covers the
>   same time and the total-versus-per-hour comparison never arises. A slot-count entry
>   point is kept for the "N cheapest slots" case, and those candidates are compared on
>   duration-weighted mean price.
> - _Start granularity_: **slot boundaries only**, stated as a documented restriction of
>   the search space rather than as a free optimization — the optimum is provably at a
>   boundary for a flat load, and that proof fails once weights are non-flat.
> - _The load's own shape_: **one calculation, `cost(window, weights)`, flat weights by
>   default.** A wave-1 caller passes nothing and gets flat selection; wave 3 passes a
>   profile curve. The tension between this change and `energy-participants` _Demand
>   declaration_ was a missing parameter, not a disagreement.
> - _Under-supply_: **always the best partial answer**, carrying requested versus granted.
>   A boiler that needs 3 h and can get 2 gets 2, not 0.
>
> Stated in _Window strategies_ here and, in full, in `define-price-providers` _Shared
> window calculations_; the curve's own storage and time base come with pack A13, whose
> home is `add-named-profiles`. What the decision does **not** settle: which bundle ships
> the shared calculation, and therefore how a wave-1 classifier depends on a wave-2
> capability. The alternatives — a slot-count API, curve-aware selection deferred to wave 3,
> silence on a shortfall — are preserved below as written.

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

> **ANSWERED — owner decision D17** (Part B rows EL-12a, EL-12b, EL-12c; 2026-08-02,
> `docs/OWNER_DECISIONS.md`), all three:
>
> - _Boundaries_: **fixed month-day ranges, user-editable**, shipped as summer
>   15 May – 15 September, winter 1 November – 20 March, transition for the rest. Those are
>   storm.house's own dates — and, being neither meteorological nor astronomical, they rule
>   the other two candidate grammars out rather than merely outranking them.
> - _Zone_: the **site's**, from core's `TimeZoneProvider`, because a season is a
>   heating-demand concept about the building.
> - _The naming date of a delivery day_: the **market's** local date in the market's zone,
>   carried on the price series and never inferred. Evidence is this corpus's own fixture:
>   `dayahead-prices.csv` runs 2023-03-23T23:00Z → 2023-03-24T22:00Z, exactly the CET
>   calendar day — not the UTC day, and not the publisher's own EET day.
>
> The first two are stated in _Seasonal window defaults_; the third is stated in
> `define-price-providers` _Delivery-day identity and the market zone_, because it is a
> property of the price series rather than of the season. The alternatives — meteorological
> or astronomical seasons, arbitrary user date ranges, market or UTC season boundaries —
> are preserved below.

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

> **ANSWERED — owner decision D17** (Part B row EL-13; 2026-08-02,
> `docs/OWNER_DECISIONS.md`). The four levels stay **purely relative**: a level is a rank
> within the planning horizon, never a claim about the absolute price — said in one
> sentence in _Level derivation_, with a scenario for an all-negative day. `Number:EnergyPrice`
> is a signed `QuantityType` with no non-negativity constraint, so nothing in core objects.
> Absolute-price behaviour, if a site wants it, reuses the escalation seam and is opt-in;
> it is not part of the ranking.
>
> The alternative below — an absolute-price condition that overrides the ranking — is
> preserved. Note the decision pack's observation that renaming "blocked" (§1, still open)
> would remove half the sting on its own. The price plane's half of this question is
> answered in `define-price-providers` design.md §4.

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

> **ANSWERED in part — owner decision D6** (2026-08-02, `docs/OWNER_DECISIONS.md`). The
> second half is settled: the engine does **not** read the published Item. Publication is an
> output of the engine's own computation, the classifier→engine path is in-process, and
> shadow mode therefore still has a level — which is exactly why the prototype's in-process
> seam is the shape the corpus intends and not an implementation detail. Stated in _The
> current level is a product of the engine's cycle_ and in _Planned schedule vs. current
> level_.
>
> **Still open: one Item or two** — whether the plan rides on the same Item as the current
> level or on a second one. openHAB's `TimeSeries` is per-item and a user sees the answer in
> their item list and their charts. Nothing in D6 decides it.
>
> **FOLLOW-UP — D23** (owner decision, 2026-08-03, `docs/OWNER_DECISIONS.md`): **who** writes
> them is now settled even though **how many** is not. Both the current-level Item and the
> planned `TimeSeries` are written by the separate publishing component that D23 splits out
> of the framework, alongside the engine's status Item — the framework that derives the
> level writes nothing at all. That makes D6's own aside ("if adopted later, publication
> becomes an output of the engine") concrete: there is now a component for it to be an output
> of. The one-Item-or-two question is inherited by that component unchanged; the three
> blockers and the rejected shapes are in `define-engine-contract` design.md §23.

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

> **ANSWERED — owner decision D6** (2026-08-02, `docs/OWNER_DECISIONS.md`), and this is the
> one place the direction is stated. **The engine computes the level from its own
> snapshot**: the level plane is a pure function the engine calls, not a service it reads.
> Rationale: it is what makes "one consistent snapshot per cycle" true — level and readings
> cannot disagree inside a cycle, and the engine never reads the grid twice. Evidence: it is
> what the prototype had to do to compile, which it reported as removing a contradiction
> rather than as an answer, precisely because the corpus never said which way the dependency
> ran.
>
> Alternative preserved, and it is a real one: the level published separately as an Item
> plus a planned `TimeSeries`, closer to masipila's planned-versus-current split, with the
> engine reading it back. If that is ever adopted, publication becomes an **input** to the
> engine rather than an output of it, the classifier and the engine couple through the item
> registry, the in-process seam disappears, and shadow mode — which writes nothing — needs
> its own answer for where the level comes from. That is a materially different
> architecture, which is why the corpus states the direction rather than leaving it to an
> implementer.
>
> Stated in _The current level is a product of the engine's cycle_. The same decision is
> recorded on the engine's side in `define-engine-contract` design.md §4 and
> `define-engine-contract` design.md §18.

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

Nor is anything on this page thread consensus **after** the 2026-08-02 answer pass. Eleven
sections (§2, §3, §4, §8 in part, §9, §10, §11, §12, §13, §14 in part, §15) now carry an
owner decision; the owner is a second production-system operator, which is the corpus's
accepted provenance class for evidence but is not agreement from #3478. Every one of them
is overturnable by pointing at an option preserved in the same section, and the cost of
doing so is a rewrite of the requirement named in the answer block. Six sections (§1, §5,
§6, §7 in part, and the open halves of §8 and §14) are still open and say so.
