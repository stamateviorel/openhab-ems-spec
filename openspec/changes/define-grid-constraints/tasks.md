# Tasks

> **Decision pass 2026-08-02.** D4, D15, D16 (pack A14), D17 and D19 answered
> design.md §1–§5 (record: `docs/OWNER_DECISIONS.md`; owner decisions in a reference implementation,
> not thread consensus). Task 1.3 is new and is the one piece of work a decision created
> rather than closed.

## 1. Alignment

- [ ] 1.1 Collect the constraint archetypes beyond BE/NO (DE §14a dimming? others?) from the community
- [ ] 1.2 Decide the delivery vehicle for regional definitions (data file vs. add-on vs. both)
- [ ] 1.3 **Derive a discriminating budget fixture from the owner's own site (D19).** Four
      EV chargers sharing an ECO solar budget under the Belgian capacity tariff is a genuine
      partial-fit vector — two loads that _can_ share a slot — which is what the existing
      pair cannot exercise (9 + 3 = 12 > 10 degenerates to mutual exclusion). Pull a real run
      from site persistence, derive the expected schedule, state the slot geometry, and add
      it to `fixtures/` with its provenance. **Do not fabricate data**: if no real run yields
      a partial fit, the honest outcome is to keep the non-discriminating note. Alternatives
      preserved in design.md §4 (ask masipila/mstormi in the thread; ship without one)

## 2. Specification hardening

- [x] 2.1 Formalize the budget/priority interaction with wave-1 priorities (one ordering, not
      two) — **decided (D4): lower number = better in both senses, default 100, ties by
      participant id ascending**. Follow-on recorded in design.md §5: the owner's live
      binding says the opposite for conflicts and needs reconciling
- [ ] 2.2 Peak-tracking semantics across restarts (persisted state) — now sharper: what must
      survive is the month-to-date billed peak and the current quarter's running average,
      both denominated in the metered quantity (D16 pack A14)
- [ ] 2.3 Re-check the shed threshold against a real bill when 3.1 lands. The corpus states
      `max(month-to-date peak, minimum billable demand)` less the margin, in **unsigned
      import watts**; pack A14 transcribed the reference's `Math.min(mtdPeak,
      −minBillableW)`, where both terms are negative under grid-positive-is-export, so the
      unsigned restatement inverts the selector (design.md §3). Both sibling requirements —
      _Established peak as budget_ and _Below the minimum billable demand there is nothing
      to shave_ — agree with `max`, and a real bill is the way to be sure

## 3. Prototype path

- [ ] 3.1 Bucket-peak tracker as a pure function with the Belgian parameters as test vectors —
      zone-local quarters at :00/:15/:30/:45, commit at rollover, monthly reset, projection
      reconciled every cycle
- [ ] 3.2 Budget scheduler over the thread's 3 kW + 9 kW example, and over task 1.3's vector
      once it exists — passing only the first proves nothing about budget enforcement
- [ ] 3.3 Replanning path end-to-end: consume a published `DEFERRED` outcome from the runtime
      floor and reschedule on the planner's own cadence, with no feedback into the deferring
      cycle
