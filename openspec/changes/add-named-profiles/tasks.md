# Tasks

> **Decision pass 2026-08-02.** D16 (pack A13) and D13 answered design.md §1–§4 (record:
> `docs/OWNER_DECISIONS.md`; owner decisions in a reference implementation, not thread
> consensus). The container question (1.1) is now sharper rather than closed: a
> relative-time shape is a small config entity, not a persisted series.

## 1. Alignment

- [ ] 1.1 Decide the profile container — D16 (pack A13) settles what it holds: a
      **relative-time shape**, offsets plus a scale factor, not an absolute `TimeSeries`, so
      it is a small config entity rather than something on the price/forecast storage plane.
      Which entity, and how it is edited, is still open (wave-1 design.md §1 settled by D5:
      metadata plus SPI behind one provider)
- [ ] 1.2 Collect 2–3 real measured program curves as reference fixtures (@jlaur's
      dishwasher mapping, a wash program, a DHW cycle) — must come from the thread or a
      production system; do not synthesize a curve

## 2. Specification hardening

- [x] 2.1 Define curve normalization (per-slot fractions × scale vs. absolute W) — **decided
      (D16 pack A13)**: samples are offsets from start, each the average power over the
      interval to the next sample, last interval ending at the declared runtime, scaled by
      the scale factor, integrated **LEFT-Riemann**; sample count capped at 1440 as a
      declaration guard with no semantic meaning
- [ ] 2.2 Profile-switch semantics mid-plan (never mid-Batch-run) — and, per D13, a switch
      never resumes a consumer marked `never`

## 3. Prototype path

- [ ] 3.1 Extend the wave-1 model with named profiles behind a flag — wave 1's implicit
      unnamed profile is the rectangular special case, so this should be one model, not a
      parallel one
- [ ] 3.2 Seasonal auto-selection over the calendar
- [ ] 3.3 Anchoring test: one stored shape scheduled at two different starts yields two
      `TimeSeries` and leaves the profile untouched
