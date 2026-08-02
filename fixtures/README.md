# Acceptance fixtures

Executable test vectors extracted **verbatim** from the thread — real data posted by the
people who run these algorithms in production. Use them as unit-test fixtures for any
implementation of the corresponding requirements.

## Provenance and integrity

All four CSVs come from @masipila's worked example in
[#3478 comment 1481931363](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931363)
(2023-03-23). They were extracted programmatically from the comment text (decimal commas
normalized to dots) and **machine-verified to be internally consistent** before inclusion:

- `expected-planned-levels.csv` exactly equals the stated rule applied to the prices
  (cheapest 4 slots → level 3, next-cheapest 4 → level 2, most-expensive 4 → level 0,
  rest → level 1).
- `expected-heating-control.csv` is exactly the 8 cheapest slots ON.
- `expected-boiler-control.csv` is exactly the next-3-best slots ON, with zero overlap
  with the heating schedule.

His hand-worked 2023 tables pass all three checks untouched — they are genuine
acceptance vectors, not illustrations.

## Slot geometry

Every row carries a slot **start** and no end. All four files share one timestamp column,
row for row, and the series is uniform: **24 slots of 60 minutes**, the first starting
`2023-03-23T23:00Z` and the last starting `2023-03-24T22:00Z`, so the series ends
`2023-03-24T23:00Z`. Stated here because it is otherwise inferable only by reusing the
previous row's width — an inference a 15-minute or mixed-width vector would make
ambiguous. A future vector should state its own width the same way.

Those bounds also name the delivery day. `2023-03-23T23:00Z` → `2023-03-24T22:00Z` is
exactly the **CET calendar day 2023-03-24** — not the UTC day, and not the publisher's own
EET day. That is the evidence behind `price-data` _Delivery-day identity and the market
zone_ (owner decision D17 · EL-12c, `docs/OWNER_DECISIONS.md`): a delivery day is named in
the market's zone, which is carried on the series and never inferred.

This is a description of the data as it stands, not a statement that slots must be
uniform or that a series may not be gapped. The corpus does permit both: `price-data`
_Time resolution_ mandates non-uniform and mixed intervals, and owner decision D16
(pack A12) settles what a gap means — contiguity is abutting, so a hole breaks a
consecutive window (`define-energy-levels` design.md §11, L13/L14/L15).

## The vectors

| File | Contents | Exercises |
|---|---|---|
| `dayahead-prices.csv` | 24 hourly day-ahead prices (ct/kWh), 2023-03-23T23:00Z → 2023-03-24T22:00Z | input for all three below |
| `expected-planned-levels.csv` | the planned level series for those prices with 4/4/4 hour counts | `energy-levels` → _Level derivation_ ("Cheapest-hours classification" scenario) |
| `expected-heating-control.csv` | 9 kW direct heating, 8 h needed, priority 1 → ON/OFF per slot | `energy-levels` → _Window strategies_ (non-consecutive selection) |
| `expected-boiler-control.csv` | 3 kW boiler, 3 h needed, priority 2, must not overlap heating (10 kW budget) | `grid-constraints` → _Load balancing under a power budget_ — with the two caveats below |

### The boiler vector's two caveats

Both were surfaced by the wave-1 prototype (E16 and L18, see
[`../docs/PROTOTYPE_FEEDBACK.md`](../docs/PROTOTYPE_FEEDBACK.md)). Neither is a reason to
distrust the data — the vector is genuine — and neither is resolved here.

**It needs look-ahead planning, which no wave-1 requirement describes.** Reproducing this
file means moving a load to a cheaper hour. A runtime limit floor can defer a load; it
cannot re-plan one. An earlier version of this table also attributed the file to
`engine-contract` → _Deterministic conflict resolution_; that attribution is withdrawn as
unsupported, because no requirement in `engine-contract` describes the replanning the file
demands. Whether that change should grow a deferral→replanning path, or whether the
behaviour belongs wholly to `grid-constraints`, was open when this note was written — both
`define-engine-contract` design.md §17 and `define-grid-constraints` design.md §4 now carry
the answer recorded immediately below, with the path not taken preserved in each.

_Ownership decided 2026-08-02 (D16 pack A14, [`../docs/OWNER_DECISIONS.md`](../docs/OWNER_DECISIONS.md))._
**`grid-constraints` owns look-ahead and replanning; `engine-contract` owns the runtime
floor and a terminal deferral-to-report path only** — the floor publishes what it refused
and the planner consumes that on its own cadence, so exactly one change claims the
behaviour this file needs. The attribution above therefore stands, and the requirement that
makes it true is `grid-constraints` _Look-ahead and replanning live with the planner_.

E16 offered two remedies: correct the attribution, or add a replanning requirement to
`engine-contract` that would make it true. This pass took the first, because the second
would answer the open question rather than record it. It is the one place in the
prototype-feedback pass where a choice between two offered remedies was made, and it is
cheap to reverse — reversing it means adding that requirement and restoring this row, not
undoing anything.

**It cannot discriminate a budget scheduler from a naive exclusion scheduler.** The two
loads draw 9 kW and 3 kW against a 10 kW budget, and 9 + 3 = 12 > 10, so under this budget
they can never share a slot: the budget constraint degenerates to mutual exclusion for
this data. The file reproduces exactly as "the next three cheapest hours, excluding the
heating hours" — which is what the prototype did — so passing it is **necessary but not
sufficient** evidence that an implementation enforces a power budget at all.

_Sourcing decided 2026-08-02, data still missing (D19,
[`../docs/OWNER_DECISIONS.md`](../docs/OWNER_DECISIONS.md))._ The owner's ruling is to
**derive a discriminating vector from their own site** — four EV chargers sharing an ECO
solar budget under a capacity tariff is a genuine partial-fit case, and a second production
system is this corpus's own accepted provenance class, so it is sourcing rather than
inventing. **Nothing has been added yet**: no run has been pulled, no expected schedule
derived, and the caveat above stands unchanged until one is. Tracked as
`define-grid-constraints` task 1.3, which says in terms that the numbers come from a real
run and must not be fabricated. Alternatives preserved in `define-grid-constraints`
design.md §4: ask masipila or mstormi in the thread, or ship without one.

## Using them

A conforming level classifier, given `dayahead-prices.csv` and hour counts
overcapacity=4 / low-price=4 / blocked=4, MUST reproduce `expected-planned-levels.csv`.
A conforming scheduler, given both loads (9 kW/8 h/prio 1, 3 kW/3 h/prio 2) and a 10 kW
budget, MUST reproduce the two control series — subject to the caveat above that doing so
does not by itself demonstrate budget enforcement. Ties, if a price ever repeats, need a
documented tie-break — this dataset contains no ties, and where that tie-break should be
stated is open (`define-energy-levels` design.md §6, L3).

Two things these files used to imply without settling; both are now stated in prose, by
the owner rather than by the thread (2026-08-02, `docs/OWNER_DECISIONS.md`).
`expected-planned-levels.csv` is still the only place in the corpus where the four levels
carry numbers, and the encoding it binds (3 = cheapest … 0 = dearest) is now **the
central encoding**, fixed as `blocked = 0` … `overcapacity = 3` in `energy-levels`
_Four-level scale_ (D12) — so this file no longer binds implementations on its own, it
agrees with a requirement, and no fixture needed rewriting. `priority` runs the other way
on purpose: lower is better, in both the order-served and wins-a-conflict senses
(D4, `energy-participants` _Priority_). A level is a magnitude, a priority is a rank; the
asymmetry is deliberate and stated, not an accident of two fixtures.

Both decisions are overturnable by pointing at an option preserved in
`define-energy-levels` design.md §2 (L1) or `define-participant-model` design.md §5.2 (A4);
if either is overturned, this file is regenerated rather than reinterpreted.

More vectors welcome — especially measured device load curves (see
`add-named-profiles` task 1.2), a 15-minute-resolution price day, a day containing
repeated or negative prices, and above all **a vector where the power budget binds
non-trivially**: two loads that could legally share a slot, or a third that only partly
fits. That last one is what would close L18. Like everything else here it has to come from
the thread or from a production system — a fixture nobody runs is an illustration, and
inventing one would be worse than the gap it fills.
