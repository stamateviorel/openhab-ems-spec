# Tasks

## 1. Alignment

- [ ] 1.1 Decide the levels/objective interaction (design.md §1) with `define-energy-levels`
- [ ] 1.2 Confirm the built-in objective set (cost / self-consumption / carbon) with the thread
- [ ] 1.3 Identify carbon-intensity sources worth first-class support (per region)

## 2. Specification hardening

- [ ] 2.1 Worked example: feed-in behaviour under each objective (design.md §4)
- [ ] 2.2 Objective-neutral naming pass across the corpus ("best", not "cheapest")

## 3. Prototype path (after 1.x)

- [ ] 3.1 Objective as a pure ranking function over series — one interface, three built-ins
- [ ] 3.2 Fixture: rerun `fixtures/dayahead-prices.csv` under a synthetic green-share series
      and document the expected placement differences
