# Tasks

## 1. Alignment
- [ ] 1.1 Confirm the 4-level scale + names with the SG-ready vocabulary (maintainers + @mstormi + @masipila)
- [ ] 1.2 Decide whether the current level is one site-global signal, or site + per-domain

## 2. Specification hardening
- [ ] 2.1 Reconcile percentile-based derivation (reference) with fixed-hour-count derivation (storm.house) — both, or one with the other as configuration?
- [ ] 2.2 Define the exact planned-series semantics on re-planning (fresh overwrites old)

## 3. Prototype path
- [ ] 3.1 Level classifier as a pure, unit-testable function over a price series
- [ ] 3.2 Publish planned series via the core TimeSeries/forecast machinery
