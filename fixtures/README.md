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

This is a description of the data as it stands, not a statement that slots must be
uniform or that a series may not be gapped. Whether the corpus permits non-uniform or
gapped series is open — see `define-energy-levels` design.md §11 (L13, L14, L15).

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
behaviour belongs wholly to `grid-constraints`, is open — `define-engine-contract`
design.md §17 and `define-grid-constraints` design.md §4.

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

## Using them

A conforming level classifier, given `dayahead-prices.csv` and hour counts
overcapacity=4 / low-price=4 / blocked=4, MUST reproduce `expected-planned-levels.csv`.
A conforming scheduler, given both loads (9 kW/8 h/prio 1, 3 kW/3 h/prio 2) and a 10 kW
budget, MUST reproduce the two control series — subject to the caveat above that doing so
does not by itself demonstrate budget enforcement. Ties, if a price ever repeats, need a
documented tie-break — this dataset contains no ties, and where that tie-break should be
stated is open (`define-energy-levels` design.md §6, L3).

Two things these files imply but do not settle. `expected-planned-levels.csv` is the only
place in the corpus where the four levels carry numbers, and the encoding it binds
(3 = cheapest … 0 = dearest) runs opposite to `priority`, where the fixtures make lower
better. Neither scale's direction is stated in prose anywhere. **Reproducing these files
does not make either encoding normative** — both are open questions
(`define-energy-levels` design.md §2, L1; `define-participant-model` design.md §5.2, A4).
An implementation that reproduces the vectors under a different published encoding is not
thereby wrong.

More vectors welcome — especially measured device load curves (see
`add-named-profiles` task 1.2), a 15-minute-resolution price day, a day containing
repeated or negative prices, and above all **a vector where the power budget binds
non-trivially**: two loads that could legally share a slot, or a third that only partly
fits. That last one is what would close L18. Like everything else here it has to come from
the thread or from a production system — a fixture nobody runs is an illustration, and
inventing one would be worse than the gap it fills.
