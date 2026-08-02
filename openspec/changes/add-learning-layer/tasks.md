# Tasks

> **Decision pass 2026-08-02.** D17 (Part B LL-0 to LL-4) answered all four alignment and
> hardening questions, and the missing design file they lived in now exists as `design.md`
> (record: `docs/OWNER_DECISIONS.md`; owner decisions in a reference implementation, not
> thread consensus). One figure is deliberately **not** derived: the 30-day retention in
> §2, kept as judgement and labelled as such in the requirement.

## 1. Alignment

- [x] 1.1 Review the "propose, don't override" guardrail — **decided (D17, LL-1): keep it,
      re-sourced from `participant-discovery` _Propose, never silently activate_ rather than
      flagged as synthesized**, with the auto-apply clause narrowed to user-configurable
      parameters a participant declares learnable, and safety-relevant limits never
      auto-applying
- [x] 1.2 Agree minimum data requirements — **decided (D17, LL-2): a per-learner contract**
      (declared items + minimum retained history), enforced by refusing to run and reporting
      why. **The thermal model's 30 days is judgement, not derived** — revisit it against a
      real site rather than treating it as settled

## 2. Specification hardening

- [x] 2.1 Define the model-quality metric surface — **decided (D17, LL-3): two numbers**, a
      normalized confidence in [0,1] for "converged" and a rolling prediction error for
      "drifting", both published as items
- [x] 2.2 Specify the profile-learning trigger — **decided (D17, LL-4)**: per completed run
      for profiles, continuous/online for the thermal model, on-demand re-learn for both,
      never a nightly batch

## 3. Prototype path

- [ ] 3.1 Extract the reference's RC estimator as a standalone, data-in/data-out candidate,
      publishing both quality figures
- [ ] 3.2 Program-curve extraction over @jlaur's persisted dishwasher data as the canonical
      fixture — a real run, not a synthesized curve
- [ ] 3.3 Prove the data contract: a learner whose inputs are not persisted refuses to run
      and reports the items and retention it needs
