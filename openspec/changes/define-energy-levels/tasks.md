# Tasks

## 1. Alignment

- [ ] 1.1 Confirm the 4-level **names** with the SG-ready vocabulary (maintainers + @mstormi + @masipila) — the spec's names and the taxonomy it cites differ (design.md §1). The numeric encoding is no longer open: fixed centrally as 0–3 by owner decision D12, with the SG-ready 1–4 correspondence stated as a named mapping (design.md §2, §3)
- [ ] 1.2 Confirm site-global with the thread — decided as site-global with a `level()` seam by owner decision D17 · EL-4 (design.md §4); what still needs agreement is that the seam's shape is right, since it is a signature, not a preference
- [ ] 1.3 Put the mode-1 SG-Ready cap to @mstormi: the corpus states it participant-side and out of scope for core, and the participant model can express "at most 2 h per assertion" but not "at most 3 assertions per day" (design.md §3)

## 2. Specification hardening

- [ ] 2.1 Decide tie handling and adaptivity for band derivation: percentile and fixed-count derivation are identical on tie-free data (proved on the corpus fixture, design.md §5), so what is open is what happens to slots tied on price, and whether band sizes follow the day's price spread or stay fixed — still open after the 2026-08-02 answer pass, which took only the encoding half of the pack that would have retired it (design.md §6)
- [ ] 2.2 Decide the unit of the band counts — hours or slots — which the window-request decision narrowed but did not settle (design.md §7). Until it is, _Level derivation_ is fully definite only on an hourly series, and its Source line says so
- [ ] 2.3 Source a shipped `encouragedFrom` default, or agree there is none. EL-8a fixed the graded shape and the 2× relation to `overcapacityFrom` but never a base number, and the reference implementation has no site-level escalation threshold to read one off — so the requirement states what an **unset** threshold means (no escalation above the planned level) rather than escalating at a number nobody chose. Supplying one is a one-line change and needs only a source (design.md §8)

## 3. Prototype path

- [ ] 3.1 Level classifier as a pure, unit-testable function over a price series, called by the engine on its own snapshot rather than read back from an Item (D6)
- [ ] 3.2 Publish planned series via the core TimeSeries/forecast machinery, re-publishing `[now, end)` with `Policy.REPLACE` on any configuration change (D17 · EL-10)
