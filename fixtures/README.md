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

## The vectors

| File | Contents | Exercises |
|---|---|---|
| `dayahead-prices.csv` | 24 hourly day-ahead prices (ct/kWh), 2023-03-23T23:00Z → 2023-03-24T22:00Z | input for all three below |
| `expected-planned-levels.csv` | the planned level series for those prices with 4/4/4 hour counts | `energy-levels` → _Level derivation_ ("Cheapest-hours classification" scenario) |
| `expected-heating-control.csv` | 9 kW direct heating, 8 h needed, priority 1 → ON/OFF per slot | `energy-levels` → _Window strategies_ (non-consecutive selection) |
| `expected-boiler-control.csv` | 3 kW boiler, 3 h needed, priority 2, must not overlap heating (10 kW budget) | `grid-constraints` → _Load balancing under a power budget_; `engine-contract` → _Deterministic conflict resolution_ |

## Using them

A conforming level classifier, given `dayahead-prices.csv` and hour counts
overcapacity=4 / low-price=4 / blocked=4, MUST reproduce `expected-planned-levels.csv`.
A conforming budget scheduler, given both loads (9 kW/8 h/prio 1, 3 kW/3 h/prio 2) and a
10 kW budget, MUST reproduce the two control series. Ties, if a price ever repeats, need
a documented tie-break — this dataset contains no ties.

More vectors welcome — especially measured device load curves (see
`add-named-profiles` task 1.2) and a 15-minute-resolution price day.
