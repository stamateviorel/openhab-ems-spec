# Tasks

## 1. Alignment

- [ ] 1.1 Review with the price-binding authors (@jlaur EnergiDataService, Tibber/ENTSO-E maintainers)
- [ ] 1.2 Decide: generic grid-price provider in core vs. as an add-on (VAT/transformation dependency question from the thread)

## 2. Specification hardening

- [ ] 2.1 Enumerate the standard adjustment set for the generic provider (VAT, unit, fixed fee, day/night, season, weekday)
- [ ] 2.2 Draft the calculation API surface fixed by owner decision D16 · pack A12 — `Duration` requests, one `cost(window, weights)` with flat weights by default, slot-boundary starts, abutting contiguity, partial answers carrying requested versus granted — and confirm it with the price-binding authors (design.md §1, §2, §3)
- [ ] 2.3 Decide which bundle ships the shared calculation, since wave-1 window selection now calls it (`define-energy-levels` design.md §11 records that the decision does not settle this)

## 3. Prototype path

- [ ] 3.1 Window calculations as pure functions with the thread's examples as test vectors, including an all-negative price day and a gapped series
- [ ] 3.2 Adapter proof: one existing price binding exposed through the new interface unchanged
