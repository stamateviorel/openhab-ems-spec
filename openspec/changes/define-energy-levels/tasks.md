# Tasks

## 1. Alignment

- [ ] 1.1 Confirm the 4-level scale + names with the SG-ready vocabulary (maintainers + @mstormi + @masipila) — the spec's names and the taxonomy it cites differ, and the scale's numeric encoding is unstated (design.md §1, §2)
- [ ] 1.2 Decide whether the current level is one site-global signal, or site + per-domain — this is a signature, not a preference (design.md §4)

## 2. Specification hardening

- [ ] 2.1 Decide tie handling and adaptivity for band derivation: percentile and fixed-count derivation are identical on tie-free data (proved on the corpus fixture, design.md §5), so what is open is what happens to slots tied on price, and whether band sizes follow the day's price spread or stay fixed
- [ ] 2.2 Define the exact planned-series semantics on re-planning (fresh overwrites old) — settle together with when rule-updated hour counts take effect (design.md §10)

## 3. Prototype path

- [ ] 3.1 Level classifier as a pure, unit-testable function over a price series
- [ ] 3.2 Publish planned series via the core TimeSeries/forecast machinery
