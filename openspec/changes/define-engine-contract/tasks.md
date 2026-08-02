# Tasks

## 1. Alignment

- [ ] 1.1 Maintainer/thread review — especially the three reference-sourced requirements
      (master stop, ACK actuation, limits-regardless-of-algorithm)
- [ ] 1.2 Decide cadence model (design.md §1) and shadow/stop granularity (§2)
- [ ] 1.3 Decide where ACK handling lives (§3) together with `define-extension-points`
- [ ] 1.4 Decide what the master stop covers — evaluation as well as dispatch (§6), what
      survives a resume (§7), and what a device protection does while it is engaged (§5).
      Three safety decisions surfaced by the wave-1 prototype (N4, E10, N5); an
      implementation must not settle any of them
- [ ] 1.5 Settle the engine-owned enumeration's membership — level gate, readiness
      interlock (§16). This is the extension surface's security boundary (N7)
- [ ] 1.6 Resolve the level-plane coupling with `define-energy-levels`: input or product
      (§4, I1) and in-process or through published Items (§18, I2)

## 2. Specification hardening

- [ ] 2.1 Define the minimum context-snapshot content (§4), including whether an algorithm
      may retain a snapshot beyond its cycle (E17), and name the source of elapsed time for
      the protection durations (N3)
- [ ] 2.2 Reconcile the runtime limit floor with `define-grid-constraints` planning
      budgets — one declaration, two consumers of it; and say whether a deferral feeds back
      into replanning (§17, E16)
- [ ] 2.3 State the trim-or-defer rule (§8) and what the floor books, estimates or
      measurements (§9); decide how mode changes are budgeted (§10) and what counts as
      surplus (§11)
- [ ] 2.4 Define the decision-outcome vocabulary (§12), the shadow-output surface (§14) and
      the handling of a decision naming an unknown participant (§13)
- [ ] 2.5 Define algorithm identity and ownership — who may replace whose id (§15)

## 3. Prototype path (after 1.x)

- [ ] 3.1 Port the reference's controller/priority contract as a candidate core interface
- [ ] 3.2 Shadow-mode harness: decisions logged + comparable against existing automation
- [ ] 3.3 Script-algorithm proof: one trivial algorithm implemented as a JSR223 script
      driving the same engine
