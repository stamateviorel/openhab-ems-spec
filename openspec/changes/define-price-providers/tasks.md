# Tasks

## 1. Alignment
- [ ] 1.1 Review with the price-binding authors (@jlaur EnergiDataService, Tibber/ENTSO-E maintainers)
- [ ] 1.2 Decide: generic grid-price provider in core vs. as an add-on (VAT/transformation dependency question from the thread)

## 2. Specification hardening
- [ ] 2.1 Enumerate the standard adjustment set for the generic provider (VAT, unit, fixed fee, day/night, season, weekday)
- [ ] 2.2 Define the calculation API surface (engine + rules + scripts access)

## 3. Prototype path
- [ ] 3.1 Window calculations as pure functions with the thread's examples as test vectors
- [ ] 3.2 Adapter proof: one existing price binding exposed through the new interface unchanged
