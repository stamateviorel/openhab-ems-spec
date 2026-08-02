# Tasks

> **Decision pass 2026-08-02.** D16 (pack A8), D17 (Part B UI-1, UI-2) and D18 (Part C
> UI-3) answered design.md §1, §2, §3 and §5 (record: `docs/OWNER_DECISIONS.md`; owner
> decisions in a reference implementation, not thread consensus). The minimum set is now a
> requirement — stated as a floor, so mstormi's "UI time-sink" warning still holds and a
> feedback round can still widen it.

## 1. Alignment

- [x] 1.1 Decide the shipping path (design.md §1) — **decided (D17, UI-1): framework-served
      via core's UI-component provider mechanism, in a bundle separate from the engine**, so
      moving into MainUI later is a deletion. The webui-native path is preserved as the
      sequenced end state
- [ ] 1.2 Agree the overview minimum set (§2) — **floor decided (D17, UI-2)** and written up
      as _Overview minimum set_; the feedback round that was meant to freeze it is still
      worth running, and can only widen the floor
- [x] 1.3 Survey persistence services for aggregation support (§3) — **closed from core's
      SPI alone (D18, UI-3)**: `FilterCriteria` has no aggregation field, so the answer is
      pre-computed rollup series, one point per day into dedicated items

## 2. Specification hardening

- [x] 2.1 Turn the agreed minimum set into requirements with scenarios — done: _Overview
      minimum set_, including the status line that carries degraded sources and declaration
      errors (D16 pack A8)
- [ ] 2.2 Define the bounded-point contract for long-range views (target point counts) — the
      mechanism is decided (daily rollups); the numbers are not

## 3. Prototype path (after 1.x)

- [ ] 3.1 Framework-served overview page from declared participants (reference pattern),
      built in its own bundle from day one
- [ ] 3.2 One chart with actuals + forecast + planned levels on a shared axis
- [ ] 3.3 Year view rendered from a rollup series; measure payload vs raw, and verify the
      day-boundary rule (snapshot before the boundary, increment from the last reading
      before the reset)
- [ ] 3.4 Render the guided-setup form and its declaration errors from the published
      `energy:` config description, with no EMS-specific UI code
